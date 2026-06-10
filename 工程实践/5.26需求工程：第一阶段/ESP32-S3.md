这里 ESP32-S3 负责：

- 给 USB 总线提供 Host 角色
- 检测 CH552G 插入
- 读取 CH552G 的 USB 描述符
- 打开 CDC-ACM 接口
- 发送 PING\r\n
- 接收 PONG\r\n

你不用一开始理解所有 USB 协议细节，但要先抓住一句话：

**Host 主动管理总线，Device 被 Host 枚举和访问。**



设备插入
设备枚举
设备打开
数据传输
设备拔出
资源释放

**3. 你现在要先学会看这几个关键词**

后面你看到 ESP-IDF 示例时，重点找这些东西：

`usb_host_install usb_host_lib_handle_events cdc_acm_host_install cdc_acm_host_open cdc_acm_host_data_tx_blocking cdc_acm_host_close`

它们分别对应：

`安装 USB Host 处理 USB 事件 安装 CDC 驱动 打开 CDC 设备 发送数据 关闭 CDC 设备`


usb_host_install

它的意思是：

> ESP32-S3 开始扮演 USB Host。

就像电脑开机后 USB 控制器开始工作。没有这一步，ESP32-S3 不会枚举 CH552G。

你要理解成：

`打开 USB Host 总控制器 准备识别插入的 USB 设备`

**第二块：USB Host 事件任务**

找这个函数：

usb_host_lib_handle_events

它通常在一个单独任务里循环运行。

它的作用是：

> 持续处理 USB 底层事件。

比如：

`设备插入 设备拔出 设备释放 Host 内部状态更新`

这个任务不是你的业务逻辑，但必须存在。你可以把它理解成 USB 系统的“后台值班任务”。

**第三块：CDC-ACM 驱动安装**

找：

cdc_acm_host_install

这一步的意思是：

> 在 USB Host 之上，加载 CDC-ACM 类驱动。

USB Host 只知道“有 USB 设备”。  
CDC-ACM 驱动才知道“这是一个 USB 虚拟串口设备”。

所以层次是：

`USB Host Library | +-- CDC-ACM Host Driver | +-- CH552G CDC Device`

**第四块：打开 CH552G 设备**

找：

cdc_acm_host_open

官方示例里打开的是 Espressif 自己的 TinyUSB 示例设备，VID/PID 不是你的 CH552G。

你的 CH552G 当前是：

`VID = 0x1A86 PID = 0x5722`

这个来自你 CH552G 的设备描述符：

`0x86, 0x1a, 0x22, 0x57`

USB 描述符里是小端格式，所以读出来是 1A86:5722。

这里你要理解：

> ESP32-S3 不是随便打开一个 USB 设备，而是按 VID/PID 找到 CH552G。

**第五块：发送数据**

找：

cdc_acm_host_data_tx_blocking

它的意思是：

> ESP32-S3 通过 CDC OUT 端点给 CH552G 发数据。


usb_host_manager = 管 USB 总线
ch552_cdc_client = 管 CH552G 这个设备
app_main = 安排谁先启动、谁后执行


首版不是为了“做完整功能”。

首版只证明一件事：

`ESP32-S3 可以替代电脑，作为 USB Host 控制 CH552G CDC 设备。`

所以首版成功标准很简单：

`ESP32-S3 Monitor: usb: host init ok cdc: open ok tx: PING rx: PONG tx: BAD rx: ERR CMD`


先解释每个文件。

**1. app_main.c**

这是入口文件。

它只负责三件事：

`初始化日志 初始化 USB Host Manager 启动当前调试阶段`

你不要在这里直接写：

`cdc_acm_host_open cdc_acm_host_data_tx_blocking PING/PONG 细节 摄像头逻辑 Hub 逻辑`

原因是：app_main.c 应该保持干净，它是“总入口”，不是“业务垃圾桶”。

你以后看到 app_main.c，应该能一眼看出系统启动顺序。

**2. usb_host_manager.c / .h**

这个模块专门管理 USB Host 底层。

它负责：

`usb_host_install 创建 usb_lib_task usb_host_lib_handle_events 设备释放 未来 Hub 支持相关的底层事件`

它不关心 CH552G，也不关心 PING。

它只回答一个问题：

`ESP32-S3 的 USB Host 系统有没有正常运行？`

你可以把它理解成：

`USB 总线管理员`

以后如果设备插拔、释放、Host 初始化出问题，优先看这个模块。

**3. ch552_cdc_client.c / .h**

这个模块专门管理 CH552G 这个 CDC 设备。

它负责：

`按 VID/PID 打开 CH552G 发送一行文本命令 接收一行文本响应 ch552_ping ch552_send_bad_test ch552_get_status`

它不负责 USB Host 安装。  
它默认 USB Host 已经由 usb_host_manager 启动好了。

你可以把它理解成：

`CH552G 设备驱动层`

它面向你的业务协议：

`PING BAD STATUS L1 ON ALL OFF`

**4. app_stage.c / .h**

这个模块负责“阶段运行”。

你的项目不能一上来就同时跑：

`CH552G Hub Camera Integration`

那样出错时不知道是谁的问题。

所以 app_stage 负责按阶段跑：

`STAGE_BOOT_LOG STAGE_CDC_DIRECT STAGE_CDC_THROUGH_HUB STAGE_UVC_THROUGH_HUB STAGE_INTEGRATION`

首版只实现：

`STAGE_CDC_DIRECT`

它的流程是：

`打开 CH552G 发送 PING 等待 PONG 发送 BAD 等待 ERR CMD 打印结果`

**5. camera_uvc_client.c / .h**

首版只放占位。

现在不写摄像头逻辑，不引入 UVC 依赖。

它存在的意义是提醒你：

`后续摄像头是独立模块，不要塞进 app_main 或 ch552_cdc_client。`

首版它可以只有很简单的占位函数，比如“未实现”的日志。你现在只需要知道它的边界：

`只管 USB 摄像头 不管 CH552G 不管 Host 初始化`

**6. project_config.h**

这个文件放项目级常量。

比如：

`CH552G VID CH552G PID CDC 读写超时时间 命令缓冲区长度 当前阶段宏`

为什么不把这些散落在各个 .c 文件里？

因为以后你改 VID/PID、超时时间、阶段选择时，不应该到处找。

**7. idf_component.yml**

这个文件负责声明 ESP-IDF 组件依赖。

首版你只需要 CDC Host 依赖：

`usb_host_cdc_acm`

先不要加 UVC。

原因：UVC 会引入更多配置、内存、摄像头格式问题。现在我们只验证 CDC 直连，别让摄像头复杂度提前进来。



安装 USB Host
创建 USB Host event task
循环调用 usb_host_lib_handle_events()
处理 USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS
处理 USB_HOST_LIB_EVENT_FLAGS_ALL_FREE


sb/usb_host.h
提供：

text


usb_host_install
usb_host_lib_handle_events
usb_host_device_free_all

USB Host 已启动
CDC-ACM Host 驱动已安装
ESP32-S3 能按 VID/PID 找到 CH552G
能打开 CH552G CDC 设备句柄



### 问题

更重要的是：你的项目正在做 **USB Host**。ESP32-S3 的原生 USB 外设一旦被程序拿去做 Host，电脑侧就看不到它作为 USB 设备的 COM 口了。所以它在 ROM 下载模式下能出现 COM，程序运行后 COM 消失，这个逻辑是说得通的。


 原生 USB，可能在下载模式出现 COM
USB HOST     给外设用，比如 CH552G / Hub / 摄像头
USB-UART0    最适合烧录和 monitor 的 UART 调试口
如果你现在插的是 USB DEV，那出现“下载模式有 COM，运行后没 COM”很正常。
你要把电脑 USB 线插到板子下方标着：


USB-UART0：烧录 + monitor 日志
USB HOST：接 CH552G / Hub / 摄像头
USB DEV：暂时不要作为主要调试口



CDC RX callback
    -> 收到原始字节
    -> 拼成一行，以 \n 作为结束
    -> 去掉 \r\n
    -> 放入 FreeRTOS queue

上层函数
    -> ch552_cdc_send_line("PING")
    -> ch552_cdc_read_line(...)
    -> 判断返回是不是 "PONG"


1. CDC line RX 机制
2. send_line / read_line / request_line
3. ping / led / all / status 封装
4. build 通过

五、两侧对称关系总结
ESP32-S3 (Host)                         CH552G (Device)
────────────────                        ────────────────
ch552_cdc_send_line("PING")             command_parser_feed(byte)
  发送一行文本                              逐字节接收解析
  拼接 \r\n                                以 \n 为分隔符
  
handle_rx(data, len)                     usb_cdc_send_line("PONG")
  逐字节接收拼装                             发送一行文本
  以 \n 切割成行                             拼接 \r\n
  放入队列给其他任务处理
两侧的设计完全对称：文本协议 + \r\n 换行分隔。



*** Device descriptor ***
bLength 18
bDescriptorType 1
bcdUSB 2.00
bDeviceClass 0xef
bDeviceSubClass 0x2
bDeviceProtocol 0x1
bMaxPacketSize0 64
idVendor 0x349c
idProduct 0x3307
bcdDevice 3.00
iManufacturer 1
iProduct 2
iSerialNumber 3
bNumConfigurations 1


UVC 回调负责“快进快出”，采集任务负责“统计、缓存、归还帧”。



这个 callback 的作用不是“处理图片”，而是：

`通知你：新的一帧来了`

它应该做的事情非常少：

`收到 frame 记录 frame 指针、长度、时间 把 frame 信息放进 queue 立刻返回`

它不应该做这些事：

`不要在 callback 里 HTTP 上传 不要在 callback 里 JPEG 解码 不要在 callback 里长时间 printf 不要在 callback 里 malloc 大内存 不要在 callback 里做复杂 memcpy 不要等待信号量很久`


UVC callback
    ↓
只把 frame 信息丢进 FreeRTOS queue
    ↓
camera capture task
    ↓
统计 FPS / 丢帧 / 超时
    ↓
复制到 latest frame cache
    ↓
归还 UVC frame buffer


所以我们设计一个很小的消息结构：

`typedef struct { uvc_host_frame_t *frame; size_t data_len; uint32_t timestamp_ms; uint32_t sequence; } camera_frame_msg_t;`

这个结构体只保存 **元信息**：

`frame：驱动给我们的帧对象指针 data_len：这一帧 JPEG 数据长度 timestamp_ms：收到这一帧的时间 sequence：第几帧`


因为这一帧最后要归还给 UVC 驱动。归还以后，驱动可能马上复用这块 buffer 存下一帧。  
所以如果你只是保存指针，后面上传任务再读它时，里面的数据可能已经变了。

正确做法是：

`把 frame->data 复制到我们自己的 latest frame cache 然后归还 UVC frame`

UVC callback 不碰 latest cache
capture task 负责写 latest cache
未来 upload task 负责读 latest cache
这一节的核心结论是：

text


不能保存 frame->data 指针；
必须复制到自己的缓存；
无 PSRAM 时只保留最新一帧；
缓存太小就丢帧并统计；
读写缓存要考虑互斥。


UVC 摄像头
   ↓
UVC frame callback
   ↓
frame queue
   ↓
capture task
   ↓
stats 统计 + latest frame cache
   ↓
未来：HTTP 上传 / 本地处理 / 云端识别


**第二层：数据流设计**

核心数据流是：

`UVC Driver ↓ frame callback ↓ frame queue ↓ capture loop / capture task ↓ latest frame cache ↓ future upload task`

这里有一个非常重要的设计原则：

`callback 不处理图片，只转交 frame。 capture task 才处理图片。`

也就是说，callback 是入口，但不是业务逻辑中心