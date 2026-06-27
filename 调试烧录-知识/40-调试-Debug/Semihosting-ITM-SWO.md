---
aliases:
  - Semihosting
  - ITM
  - SWO
  - 调试跟踪
  - printf 重定向
tags:
  - 调试/知识体系
  - 调试/跟踪
  - Cortex-M
date: 2026-06-25
status: 🌿草稿
---

> [!abstract] 核心本质
> Semihosting / ITM / SWO 是三种**"不占用串口也能输出调试日志"**的技术。它们的共同点：日志通过**调试接口**（SWD/JTAG）送到 PC，不占 UART。区别在**机制与速度**：Semihosting 软件实现（慢，靠 BKPT 陷阱）；ITM 硬件实现（快，靠专用跟踪单元）；SWO 是 ITM 的物理输出线。调试时 printf，这三者比串口优雅得多。

---

## 一、为什么不用串口打印日志

```mermaid
flowchart LR
    subgraph 串口["❌ 串口 printf 的问题"]
        A1["占用一个 UART"]
        A2["需要接 USB-TTL"]
        A3["波特率限速(115200慢)"]
        A4["中断里 printf 有风险"]
    end
    subgraph 跟踪["✅ 调试接口输出"]
        B1["不占外设"]
        B2["只需调试器(已有)"]
        B3["ITM 高速"]
        B4["可与断点同步"]
    end
    串口 -.->|"进化"| 跟踪

    style 串口 fill:#ffebee
    style 跟踪 fill:#e8f5e9
```

> [!question] 三者到底啥关系
> - **Semihosting** = 让 MCU 借调试器调用 PC 的文件/控制台系统调用（最重，最兼容）
> - **ITM** = Cortex-M 内的硬件日志单元，往专用通道写数据（轻量、快）
> - **SWO** = ITM 数据出来的**那根物理线**（1 根引脚，复用 SWD 接口）

---

## 二、三者全景对比

| 维度 | Semihosting | ITM | 串口(UART) |
|------|-------------|-----|-----------|
| **实现** | 软件(BKPT 陷阱) | 硬件(ITM 单元) | 硬件(UART) |
| **速度** | 慢(每次陷进调试器) | 快(MHz 级) | 中(受波特率限制) |
| **占用引脚** | 0(复用 SWD) | 1(SWO 引脚) | 2(TX/RX) |
| **需要调试器** | ✅ 必须 | ✅ 必须 | ❌ 不需要 |
| **可脱机运行** | ❌(无调试器会卡死) | ❌ | ✅ |
| **中断里用** | ❌(会死锁) | ✅ 安全 | ⚠️(看实现) |
| **支持芯片** | 通用(需库支持) | Cortex-M3+ | 所有 |
| **典型场景** | 早期开发/无串口 | 实时日志 | 量产/现场 |

> [!danger] Semihosting 的致命陷阱
> Semihosting 靠 `BKPT` 指令触发调试器接管。**如果代码里有 Semihosting 但没连调试器**，`BKPT` 触发 HardFault → 死机。所以 Semihosting 代码**绝不能进量产**，必须用宏隔离。

---

## 三、Semihosting：借调试器调 PC 系统调用

### 3.1 原理

```mermaid
sequenceDiagram
    participant M as MCU 代码
    participant D as 调试器(ST-Link)
    participant PC as PC 控制台

    M->>M: printf("hi") 内部触发 BKPT
    Note over M: BKPT 让 CPU 暂停
    M->>D: 半主机请求(写"hi"到控制台)
    D->>PC: 调用 PC 的 stdout
    PC-->>D: 打印 "hi"
    D-->>M: 恢复 CPU 执行
```

> [!tip] 类比
> Semihosting 像**让保安帮你打电话**：你(MCU)不会打电话(没有串口)，但你喊一声(BKPT)，保安(调试器)就用他的电话(PC 控制台)帮你打。代价是每次都要喊保安，慢。

### 3.2 使用方式

```c
// 用 newlib 的 semihosting，初始化 retarget
extern void initialise_monitor_handles(void);
initialise_monitor_handles();   // 重定向 stdin/stdout 到半主机

printf("hello from semihosting\n");   // 输出到调试器控制台
```

链接时需 `-specs=rdimon.specs`（半主机版 libc），而非 `nosys.specs`。

### 3.3 何时用

- 没有可用串口引脚时
- 需要文件读写（半主机支持 `fopen/fread` 读写 PC 文件）
- **绝不用于中断或量产**

---

## 四、ITM：硬件级高速日志（Cortex-M3+ 推荐）

### 4.1 原理

```mermaid
flowchart LR
    CODE["代码<br/>ITM_SendChar()"] --> STIM["ITM Stimulus 端口<br/>(32个通道)"]
    STIM --> ITM["ITM 单元<br/>(硬件打包)"]
    ITM --> SWO["SWO 引脚<br/>(高速串行输出)"]
    SWO --> DBG["调试器<br/>(J-Link/ST-Link)"]
    DBG --> IDE["IDE 日志窗口"]

    style ITM fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style SWO fill:#fff9c4
```

### 4.2 ITM 的优势

- **硬件实现**：不中断 CPU 流水线，不影响时序
- **多通道**：32 个 Stimulus 端口，可区分不同模块日志
- **中断安全**：可在 ISR 里用（不像 Semihosting）
- **可与断点/变量观察同步**：ITM 带时间戳

### 4.3 使用

```c
// 通过 ITM 端口0 输出一个字符
#define ITM_Port32(n) (*((volatile unsigned int *)(0xE0000000+4*n)))

void ITM_SendChar(char c) {
    while (ITM_Port32(0) == 0);   // 等端口就绪
    ITM_Port32(0) = c;
}

// printf 重定向到 ITM
int fputc(int ch, FILE *f) {
    ITM_SendChar(ch);
    return ch;
}
```

> [!tip] J-Link 的 RTT 更香
> J-Link 有自己的 **RTT（Real-Time Transfer）**，比 ITM 更快、配置更简单。如果你用 J-Link，优先 RTT；用 ST-Link 则用 ITM/SWO。

---

## 五、SWO：ITM 的物理输出线

### 5.1 SWO 是什么

```mermaid
flowchart TB
    subgraph 调试接口["标准 SWD 接口(4线)"]
        SWCLK["SWCLK"]
        SWDIO["SWDIO"]
        SWO["SWO ★(跟踪输出,可选)"]
        GND["GND"]
    end
    subgraph 数据["SWO 承载"]
        D1["ITM 日志数据"]
        D2["PC 采样(程序计数器跟踪)"]
        D3["变量读写跟踪"]
    end
    SWO --> 数据

    style SWO fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

> [!important] SWO 的关键
> SWO 是**第 3 根线**（除 SWCLK/SWDIO 外），复用 Cortex-M 的 **TRACEWRK/TRACEDATA** 或专用引脚。STM32 上 SWO 通常是 **PB3**（或可重映射）。
>
> 不是所有调试器都支持 SWO：**J-Link、ST-Link V3、DAP-Link（部分）支持；廉价 ST-Link V2 克隆版可能不支持。**

### 5.2 SWO 采样模式

| 模式 | 时钟 | 要求 |
|------|------|------|
| **Manchester 编码** | 自带时钟 | 兼容性好，速度低 |
| **UART/NRZ 编码** | 需配 SWO 速率 = CPU 时钟/N | 速度快，需精确配速 |

---

## 六、配置流程（以 ITM+SWO 为例）

```mermaid
flowchart TD
    A["① 硬件: 连接 SWO 引脚<br/>(STM32=PB3)到调试器"] --> B["② 调试器: 启用 SWO 采集<br/>(设 SWO 速率)"]
    B --> C["③ MCU: 使能 ITM<br/>(DEMCR.TRACENA=1)"]
    C --> D["④ MCU: 设 SWO 编码/分频<br/>(TPIU)"]
    D --> E["⑤ 代码: 重定向 printf 到 ITM"]
    E --> F["⑥ IDE: 打开 SWV/Trace 窗口看日志"]

    style F fill:#e8f5e9
```

### STM32 关键寄存器配置

```c
#define DEMCR    (*(volatile uint32_t *)0xE000EDFC)
#define ITM_LAR  (*(volatile uint32_t *)0xE0000FB0)

void ITM_Init(void) {
    DEMCR |= (1 << 24);            // TRACENA 使能跟踪
    ITM_LAR = 0xC5ACCE55;          // 解锁 ITM
    // 配置 Stimulus 端口0 使能...
}
```

---

## 七、选型决策

```mermaid
flowchart TD
    START["要调试输出日志"] --> Q1{有没有空闲串口?}
    Q1 -->|"有"| S1["直接用串口<br/>最简单通用"]
    Q1 -->|"没有/想更快"| Q2{什么调试器?}

    Q2 -->|"J-Link"| J1["RTT(最快最简单)"]
    Q2 -->|"ST-Link V3/DAP"| S2["ITM + SWO"]
    Q2 -->|"廉价ST-Link V2"| Q3{支持SWO?}
    Q3 -->|"是"| S3["ITM + SWO"]
    Q3 -->|"否"| SH["Semihosting<br/>(仅开发期)"]

    style S1 fill:#e8f5e9
    style J1 fill:#e8f5e9
    style S2 fill:#e8f5e9
```

---

## 八、避坑清单

> [!warning] 跟踪技术常见坑
> 1. **Semihosting 进了量产** — 无调试器时 `BKPT` 触发 HardFault，务必用 `#ifdef DEBUG` 隔离
> 2. **SWO 速率配错** — NRZ 模式下 SWO 速率必须与 CPU 时钟匹配，否则乱码
> 3. **廉价 ST-Link 不支持 SWO** — 克隆版 ST-Link V2 经常砍了 SWO，输出无反应
> 4. **ITM 在 Cortex-M0 不可用** — ITM 需要 **M3/M4/M7**，M0/M0+ 只有简化版
> 5. **中断里 Semihosting 死锁** — Semihosting 暂停 CPU，中断里用会永久卡住
> 6. **SWO 引脚被复用** — PB3 默认是 JTAG 的 JTDO，要先释放

---

## 🔗 知识延伸

- ⬆️ **上位知识**：[[_MOC-开发流水线总览]]、[[调试全景数据流]]
- ➡️ **平级关联**：[[SWD与JTAG协议]]（SWO 复用 SWD 接口）、[[探针对比]]（谁支持 SWO）、[[HardFault排查实战]]（ITM 可安全输出崩溃日志）
- ⬇️ **下位知识**：J-Link RTT、ETM（指令跟踪，比 ITM 更全）、DEMCR/TPIU 寄存器详解
