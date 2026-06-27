---
aliases:
  - ESP32 日志调试
  - ESP32 panic 排查
  - idf.py gdb
tags:
  - 调试/知识体系
  - 调试/ESP32
  - ESP32
date: 2026-06-25
status: 🌿草稿
---

> [!abstract] 核心本质
> ESP32 的调试分两条主线：**日志驱动**（ESP_LOG + 串口监视器，覆盖 90% 日常问题）与 **崩溃定位**（panic backtrace + GDB，处理死机重启）。串口日志是第一工具，JTAG/GDB 是深水区。

---

## 一、日志输出：ESP_LOG 系列（最常用）

```c
#include "esp_log.h"

static const char *TAG = "MY_APP";

void app_main(void)
{
    ESP_LOGE(TAG, "错误信息");      // Error - 红色，总是输出
    ESP_LOGW(TAG, "警告信息");      // Warning - 黄色
    ESP_LOGI(TAG, "普通信息");      // Info - 绿色
    ESP_LOGD(TAG, "调试信息");      // Debug - 默认不显示
    ESP_LOGV(TAG, "详细信息");      // Verbose - 默认不显示
}
```

### 运行时动态调整日志等级

```c
esp_log_level_set("*", ESP_LOG_DEBUG);         // 所有组件输出 DEBUG 以上
esp_log_level_set("wifi", ESP_LOG_WARN);       // Wi-Fi 只输出 WARN 以上
esp_log_level_set("MY_APP", ESP_LOG_VERBOSE);  // 你的模块输出全部
```

### menuconfig 调试相关开关

```bash
idf.py menuconfig
```

- 更多调试信息：`Component config → Log output → Default log verbosity → Debug/Verbose`
- 看门狗打印（调试随机重启）：`Component config → Task Watchdog → Enable`
- 堆内存统计：`Component config → Heap Memory Debugging → Enable`

---

## 二、运行时查看内存与栈状态

```c
#include "esp_system.h"
#include "esp_heap_caps.h"

void app_main(void)
{
    // 可用堆内存
    ESP_LOGI(TAG, "Free heap: %d bytes", esp_get_free_heap_size());

    // 内部 SRAM 可用
    ESP_LOGI(TAG, "Free internal: %d bytes",
             heap_caps_get_free_size(MALLOC_CAP_INTERNAL));

    // PSRAM 可用
    ESP_LOGI(TAG, "Free PSRAM: %d bytes",
             heap_caps_get_free_size(MALLOC_CAP_SPIRAM));

    // 任务栈剩余（防止栈溢出）
    ESP_LOGI(TAG, "Stack HWM: %d bytes",
             uxTaskGetStackHighWaterMark(NULL));

    // 任务列表
    char buf[1024];
    vTaskList(buf);
    printf("Task\t\tState\tPrio\tStack\tNum\n");
    printf("%s\n", buf);
}
```

---

## 三、崩溃定位：panic 与 backtrace

当芯片崩溃重启时，串口会输出类似：

```
Guru Meditation Error: Core  0 panic'ed (LoadProhibited)
Backtrace: 0x400D1234:0x3FFB1234 0x400D5678:0x3FFB1250 ...
```

### 解码 backtrace

```bash
idf.py monitor
# IDF 监视器会自动将地址解码为函数名和文件行号

# 或者手动解码：
python $IDF_PATH/tools/idf_monitor.py --decode-panic build/my_project.elf
```

> [!tip] 原理
> panic 时硬件自动把寄存器压栈并打印 backtrace 地址串。每个地址形如 `PC:SP`。解码需要带符号的 `.elf` 文件（参见 [[调试全景数据流]] 中"符号文件必须匹配"）。

---

## 四、GDB 调试（JTAG，深水区）

```bash
# 启动 JTAG 调试（需要额外硬件如 ESP-Prog）
idf.py openocd
idf.py gdb
```

### 常用 GDB 命令

| 命令 | 功能 |
| --- | --- |
| `break app_main` | 设置断点 |
| `continue` | 继续运行 |
| `step` | 单步执行（进入函数） |
| `next` | 单步执行（跳过函数） |
| `print variable` | 打印变量值 |
| `backtrace` | 查看调用栈 |
| `info registers` | 查看寄存器 |

> [!warning] ESP32 的 JTAG
> ESP32 通过 JTAG 调试，协议与 STM32 的 SWD 不同（详见 [[SWD与JTAG协议]]）。需要 ESP-Prog 或带 JTAG 的下载器，普通 USB-TTL 只能烧录不能调试。

---

## 🔗 知识延伸

- ⬆️ **上位知识**：[[_MOC-开发流水线总览]]、[[调试全景数据流]]
- ➡️ **平级关联**：[[基础指令]]（烧录/环境命令）、[[GDB调试命令手册]]、[[SWD与JTAG协议]]
- ⬇️ **下位知识**：[[HardFault排查实战]]（ARM 版崩溃定位，原理相通）
