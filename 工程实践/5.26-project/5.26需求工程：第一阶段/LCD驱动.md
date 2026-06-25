你先理解三个层次：

`ST7789 驱动 ↓ Espressif BSP: esp32_s3_usb_otg ↓ 我们自己的 board_display.c/.h`

我们不直接写 ST7789 SPI 时序，因为 BSP 已经帮你处理了：

`SPI 初始化 LCD DC/CS/RST 引脚 ST7789 panel 创建 背光 PWM 屏幕分辨率 240x240`

board_display_init()
       │
       ├─ bsp_display_new()          硬件层初始化
       │    ├── 配置 SPI 总线
       │    ├── 初始化 LCD 驱动 IC（ST7789 等）
       │    ├── 分配 DMA 传输缓冲区
       │    └── 返回 panel / panel_io 句柄
       │
       ├─ esp_lcd_panel_disp_on_off()  开启像素输出
       │    └── LCD IC 收到 DISPON 指令，开始输出画面
       │
       ├─ bsp_display_backlight_on()   点亮背光
       │    └── GPIO 拉高，背光 LED 亮
       │
       └─ s_display_ready = true       标记就绪
bsp_display_new()
    初始化 ST7789 和 SPI，但不自动点亮屏幕

esp_lcd_panel_disp_on_off(s_panel, true)
    打开 LCD 显示输出

bsp_display_backlight_on()
    打开背光，否则屏幕可能黑着




这 6 个字段已经足够表达第一阶段闭环。

你后面写代码时要记住一个原则：

`board_display 不主动调用 camera_uvc_client board_display 不主动调用 ch552_cdc_client board_display 不主动调用 cloud_http_client`

它只接收别人传给它的状态，然后画出来。

这就叫**显示层和业务层解耦**。这样以后你想换 LCD 样式，不会影响摄像头、HTTP、灯控逻辑。



理解 LVGL 在这个项目里只承担“文字状态面板”
不承担摄像头实时预览
不承担业务判断
不直接调用 Camera / CH552G / HTTP



不要每次刷新都重新创建 label。  
正确做法是：

`第一次： create labels 之后： lv_label_set_text(...) lv_obj_set_style_text_color(...)`



后面 board_display_init() 的逻辑会变成：

`1. 初始化 LCD panel 2. 打开 LCD 显示 3. 打开背光 4. 初始化 LVGL port 5. 把 LCD panel 注册给 LVGL 6. 创建状态页面`

大概是：

`board_display_init() { bsp_display_new(...); esp_lcd_panel_disp_on_off(...); bsp_display_backlight_on(); lvgl_port_init(...); lvgl_port_add_disp(...); board_display_create_status_page(); }`



## . 最关键：LVGL lock

ESP32 上 LVGL 有自己的任务和互斥锁。

所以以后更新文字时，不能随便直接：

`lv_label_set_text(...);`

而是要：

`if (lvgl_port_lock(100)) { lv_label_set_text(...); lvgl_port_unlock(); }`

你可以理解为：

`我要改 LCD UI -> 先拿锁 -> 修改 label -> 放锁`

为什么？

因为 LVGL 自己的刷新任务也在访问这些 UI 对象。  
如果你不加锁，可能出现：

`显示异常 随机崩溃 内存错误 偶发 panic`


LCD 状态页 = LVGL label 页面
初始化时创建 label
运行时只更新 label
更新前必须拿 LVGL lock


初始化后的运行模型
app_main()
  ↓
board_display_init()          ← 你调用的初始化
  ↓
lvgl_port_init()              ← 创建 "lv_port" FreeRTOS 任务
  ↓ (后台持续运行)
┌─────────────────────────────────────────┐
│  lv_port 任务 (LVGL 主循环):             │
│                                          │
│  while(1) {                              │
│    lv_task_handler()                     │
│      → 检查是否有脏区域需要重绘           │
│      → 调用控件回调更新内容               │
│      → 渲染脏区域到缓冲区                 │
│      → 调用 flush_cb → SPI DMA → LCD     │
│    vTaskDelay(5ms)                       │
│  }                                       │
└─────────────────────────────────────────┘
其他任务 (app_state, http_upload, ...):
  → board_display_set_label()  更新文字
  → 只是修改 LVGL 控件属性
  → 实际渲染由 lv_port 任务异步完成
关键点：你调用 lv_label_set_text() 时不会立即刷新屏幕，只是标记"这个区域脏了"。LVGL 后台任务在下一个周期自动检测、渲染、通过 SPI DMA 刷新到 LCD。


用 LVGL 注册 LCD display，在当前 screen 上创建多个 label，用 lock 保护更新，每次只改文字和颜色。


s_system_label = board_display_create_label(screen, BOARD_DISPLAY_ROW_START_Y + 0 * BOARD_DISPLAY_ROW_STEP_Y);
s_camera_label = board_display_create_label(screen, BOARD_DISPLAY_ROW_START_Y + 1 * BOARD_DISPLAY_ROW_STEP_Y);
s_ch552_label  = board_display_create_label(screen, BOARD_DISPLAY_ROW_START_Y + 2 * BOARD_DISPLAY_ROW_STEP_Y);
公式是：

第 N 行 y = ROW_START_Y + N * ROW_STEP_Y
现在：

System: 42
Camera: 70
CH552:  98
Detect: 126
LED:    154
Cloud:  182


只要 HTTP upload task 成功过一次，就显示 Cloud OK
否则 Cloud FAIL



CAMERA_LATEST_FRAME_MAX_SIZE
96KB -> 32KB

preferred UVC:
number_of_frame_buffers = 2
number_of_urbs = 3 -> 2
urb_size = 4KB -> 2KB

fallback UVC:
number_of_frame_buffers = 1
number_of_urbs = 2
urb_size = 2KB -> 1KB