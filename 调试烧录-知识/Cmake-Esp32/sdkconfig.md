sdkconfig 文件在 ESP32 烧录中的作用
一、它到底是什么？
sdkconfig 是 ESP-IDF 构建系统的核心配置文件，本质是一个 键值对（Kconfig）文本文件，决定了：
你的固件到底编译了什么功能、占多大空间、跑在什么参数上
它 不直接参与烧录，但 决定了烧录进芯片的固件长什么样。
---
二、从原理上理解：ESP32 固件是怎么生成的
sdkconfig（你的配置）
    │
    ▼
┌─────────────────────────────────────────────┐
│          ESP-IDF 构建流程（CMake + Ninja）    │
│                                              │
│  sdkconfig                                   │
│    ├── 决定 #define 哪些宏                    │
│    ├── 决定编译哪些源文件（条件编译）           │
│    ├── 决定链接哪些库                         │
│    ├── 决定分区表布局                          │
│    └── 决定 Bootloader 参数                   │
│                                              │
│         ↓  编译  ↓                           │
│                                              │
│  输出三个文件：                               │
│  ├── bootloader.bin    → 烧录到 0x1000       │
│  ├── partitions.bin    → 烧录到 0x8000       │
│  └── your_app.bin      → 烧录到 0x10000      │
└─────────────────────────────────────────────┘
    │
    ▼
  esptool.py 烧录到 Flash
---
三、sdkconfig 里面的关键配置项（ESP32-CAM 相关）
1. 芯片目标
CONFIG_IDF_TARGET="esp32"          # 告诉编译器：为 ESP32 编译
2. Flash 与分区（直接影响烧录地址）
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y   # Flash 大小 = 4MB（你的板子）
CONFIG_ESPTOOLPY_FLASHSIZE="4MB"
CONFIG_PARTITION_LAYOUT_CONFIG="custom"  # 分区表方案
这决定了 partition table 怎么分：
4MB Flash 分区布局示例（Huge APP 方案）：
┌──────────┬───────────┬─────────────────────┐
│ 偏移地址  │ 大小       │ 用途                │
├──────────┼───────────┼─────────────────────┤
│ 0x0000   │ 3KB       │ Bootloader          │
│ 0x8000   │ 3KB       │ Partition Table     │
│ 0x10000  │ 3MB       │ Application（你的固件）│
│ 0x310000 │ 1MB       │ SPIFFS（文件系统）   │
└──────────┴───────────┴─────────────────────┘
3. PSRAM 配置（ESP32-CAM 必须开！）
CONFIG_ESP32_SPIRAM_SUPPORT=y      # 启用 PSRAM 支持
CONFIG_SPIRAM_USE_MEMMAP=y          # PSRAM 映射方式
CONFIG_SPIRAM_MALLOC_RESERVE_INTERNAL=32768  # 保留给内部堆的大小
> 不开 PSRAM = 摄像头帧缓冲只能放内部 SRAM = UXGA 直接爆内存崩溃
4. 摄像头相关
CONFIG_CAMERA_CORE0=y              # 摄像头任务跑在 Core 0
CONFIG_CAMERA_TASK_STACK_SIZE=4096 # 摄像头任务栈大小
5. Wi-Fi / BLE
CONFIG_ESP32_WIFI_STATIC_RX_BUFFER_NUM=10   # Wi-Fi 接收缓冲数量
CONFIG_ESP32_WIFI_DYNAMIC_RX_BUFFER_NUM=32
CONFIG_BT_ENABLED=y                          # 是否编译蓝牙（不用就关，省空间）
6. FreeRTOS
CONFIG_FREERTOS_HZ=1000             # tick 频率 = 1000Hz（1ms 一个 tick）
CONFIG_FREERTOS_MAX_TASK_NAME_LEN=16
CONFIG_FREERTOS_TASK_FUNCTION_WRAPPER=y
7. 串口与日志
CONFIG_ESP_CONSOLE_UART_NUM=0       # 控制台串口 = UART0
CONFIG_ESP_CONSOLE_UART_BAUDRATE=115200  # 波特率
CONFIG_LOG_DEFAULT_LEVEL_INFO=y     # 日志等级
8. CPU 频率
CONFIG_ESP32_DEFAULT_CPU_FREQ_240=y  # CPU 跑 240MHz
CONFIG_ESP32_DEFAULT_CPU_FREQ="240"
---
四、sdkconfig 是怎么生成的？
方式一：menuconfig（图形化配置）
┌─────────────────────────────────────────┐
│  idf.py menuconfig                       │
│                                          │
│  会弹出一个 ncurses 界面：               │
│  ┌──────────────────────────────────┐   │
│  │ Serial flasher config  --->      │   │
│  │ Partition Table  --->            │   │
│  │ Component config  --->           │   │
│  │   → ESP32-specific  --->         │   │
│  │     → PSRAM  --->               │   │
│  │   → Camera  --->                │   │
│  │   → Wi-Fi  --->                 │   │
│  └──────────────────────────────────┘   │
│                                          │
│  修改后保存，自动写入 sdkconfig           │
└─────────────────────────────────────────┘
方式二：直接编辑 sdkconfig 文本
  用文本编辑器直接改，然后重新编译
方式三：sdkconfig.defaults（项目默认配置）
  项目根目录放一个 sdkconfig.defaults
  首次构建时自动生成 sdkconfig
  团队协作时用它同步配置
---
五、它和烧录命令的关系
idf.py -p COM3 flash monitor
这条命令背后发生了什么：
1. CMake 读取 sdkconfig
2. 生成 build/config/sdkconfig.h（C 语言头文件，供 #ifdef 使用）
3. 根据配置选择编译源文件
4. 编译出 .bin 文件
5. esptool.py 根据 sdkconfig 中的 Flash 大小、模式、频率设置烧录参数：
   - CONFIG_ESPTOOLPY_FLASHSIZE → 告诉 esptool Flash 是 4MB
   - CONFIG_ESPTOOLPY_FLASHMODE "DIO" → Flash 读取模式
   - CONFIG_ESPTOOLPY_FLASHFREQ "40M" → Flash 频率 40MHz
6. 按分区表偏移地址写入三个 .bin
---
六、实战：ESP32-CAM 推荐的 sdkconfig.defaults
你在项目中创建这个文件，确保关键配置不遗漏：
# Target
CONFIG_IDF_TARGET="esp32"
# Flash
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_40M=y
# PSRAM - 必须开启
CONFIG_ESP32_SPIRAM_SUPPORT=y
CONFIG_SPIRAM_BOOT_INIT=y
CONFIG_SPIRAM_USE_MALLOC=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=16384
CONFIG_SPIRAM_MALLOC_RESERVE_INTERNAL=32768
# Partition - 大 APP 分区
CONFIG_PARTITION_TABLE_HUGE_CSV=y
# Wi-Fi
CONFIG_ESP32_WIFI_STATIC_RX_BUFFER_NUM=16
CONFIG_ESP32_WIFI_DYNAMIC_RX_BUFFER_NUM=32
# CPU
CONFIG_ESP32_DEFAULT_CPU_FREQ_240=y
# Camera JPEG Quality
CONFIG_CAMERA_JPEG_QUALITY=12
# Log
CONFIG_LOG_DEFAULT_LEVEL_INFO=y
---
七、避坑清单
#	坑	说明
1	改了 sdkconfig 但没重新编译	必须执行 idf.py fullclean 再编译，否则增量编译可能不刷新
2	PSRAM 没开就初始化摄像头	100% 会 OOM（Out of Memory）崩溃
3	Flash 大小配错	配成 2MB 但实际是 4MB → 固件能跑但分区表不对，可能覆盖数据
4	sdkconfig 冲突	多个 CONFIG_XXX=y 和 CONFIG_XXX=n 同时存在 → 以最后一个为准，容易混乱
5	团队成员配置不一致	用 sdkconfig.defaults 统一管理，不要直接交换 sdkconfig
6	Arduino 环境没有 sdkconfig	Arduino IDE 通过菜单选项（Tools）等价配置，底层原理相同
---