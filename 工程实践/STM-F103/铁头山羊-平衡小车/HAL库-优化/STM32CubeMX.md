---
aliases:
  - CubeMX 配置
  - STM32CubeMX 教程
  - CubeMX 参数
tags:
  - STM32
  - 平衡小车
  - CubeMX
  - HAL
  - 工具使用
related:
  - "[[工程经验]]"
  - "[[文件架构思路]]"
  - "[[STM32-Copy-car/Copy_car_Iron-headed_goat/HAL-CAR/HAL-Car/HAL-Car.ioc]]"
date: 2026-05-19
status: 整理完成
---

# STM32CubeMX 配置页面认知

> [!abstract] 实战场景
> STM32CubeMX 是 STM32 的图形化配置工具。选好芯片后，在界面上配置引脚和外设参数，它会自动生成初始化代码。本文档基于平衡车项目（STM32F103C8T6）的实际 `.ioc` 配置，逐页讲解 CubeMX 中每个参数的含义和选择依据。目标不是讲原理，而是**打开 CubeMX 看到这个参数，知道它是什么、该填什么、填错了会怎样**。

> [!note] 快速结论
> - CubeMX 的核心工作流：**Pinout & Configuration（配引脚和外设）→ Clock Configuration（配时钟树）→ Project Manager（配工程选项）→ Generate Code**。
> - 大多数参数保持默认就行，只有和你的硬件设计相关的参数才需要改。
> - 改完参数后重新生成代码，**只在 `USER CODE BEGIN/END` 区域内写的代码不会被覆盖**。

---

## CubeMX 界面总览

```
顶部标签页：
  ┌──────────────────┬───────────────────┬────────────────┐
  │ Pinout & Config  │ Clock Config      │ Project Manager│
  │ 配引脚和外设参数    │ 配时钟树            │ 配工程选项       │
  └──────────────────┴───────────────────┴────────────────┘

Pinout & Configuration 页面布局：
  ┌─────────────┬──────────────────────────────────┐
  │ 左侧面板     │  右侧芯片引脚图 / 参数面板          │
  │             │                                   │
  │ System Core │  点左侧某个外设后，下方弹出参数面板    │
  │  ├ SYS      │  ┌─────────────────────────────┐  │
  │  ├ RCC      │  │ Parameter Settings           │  │
  │  ├ GPIO     │  │ 参数配置页                     │  │
  │  ├ NVIC     │  ├─────────────────────────────┤  │
  │  ├ ...      │  │ GPIO Settings                │  │
  │             │  │ 引脚配置页                     │  │
  │ Peripherals │  ├─────────────────────────────┤  │
  │  ├ USART2   │  │ NVIC Settings                │  │
  │  ├ TIM1     │  │ 中断配置页                     │  │
  │  ├ ADC1     │  └─────────────────────────────┘  │
  │  ├ ...      │                                   │
  └─────────────┴──────────────────────────────────┘
```

> [!tip] 操作习惯
> - 左侧面板点开外设后，右侧芯片图上对应的引脚会变绿（已使用）。
> - 在芯片图上直接点击引脚，可以选择它的功能（GPIO/EXTI/TIM_CH 等）。
> - 下方 Parameter 页面改参数后，不会自动保存，需要 Ctrl+S 或点 Generate Code。

---

## 1. SYS —— 系统配置

> CubeMX 位置：System Core → SYS

### Debug 选项

| 参数 | 可选项 | 含义 | 项目值 |
|------|--------|------|--------|
| Debug | Disable / Serial Wire / JTAG(4/5 pin) | 调试接口模式 | **Serial Wire** |

```text
Serial Wire（SWD）：
  → 只用 2 根线：SWDIO(PA13) + SWCLK(PA14)
  → 最常用的调试方式，ST-Link、J-Link 都支持
  → 占用引脚最少

JTAG：
  → 需要 4-5 根线
  → 占引脚多，F103C8T6 引脚本来就紧张

Disable：
  → 不调试？烧录后无法再下载
  → 如果不小心选了这个，需要按住 Reset 再连接 ST-Link 才能恢复

⚠️ 你的项目 PB3 用于右编码器 EXTI
   PB3 默认是 JTAG 的 JTDO 引脚
   选 Serial Wire 后 PB3 释放给 GPIO 使用
   如果选 JTAG，PB3 会被占用，编码器就无法工作
```

### Timebase Source

| 参数 | 可选项 | 含义 | 项目值 |
|------|--------|------|--------|
| Timebase Source | SysTick / TIM1~TIM17 | HAL 库的时间基准（HAL_GetTick 的来源） | **SysTick** |

```text
SysTick：
  → Cortex-M3 内置的 24 位倒计时定时器
  → HAL 默认用它产生 1ms 中断，驱动 HAL_GetTick()
  → 大多数裸机项目都用 SysTick，简单够用

什么时候需要换？
  → 跑 RTOS 时，SysTick 被 RTOS 占用，HAL 需要换一个 Timer
  → 你的项目是裸机，保持 SysTick 就行
```

---

## 2. RCC —— 时钟配置

> CubeMX 位置：System Core → RCC

### 参数页

| 参数 | 可选项 | 含义 | 项目值 |
|------|--------|------|--------|
| High Speed Clock (HSE) | Disable / BYPASS / Crystal/Ceramic Resonator | 外部高速时钟来源 | **Crystal/Ceramic Resonator** |
| Low Speed Clock (LSE) | Disable / BYPASS / Crystal/Ceramic Resonator | 外部低速时钟（RTC 用） | **Disable** |
| Master Clock Output (MCO) | Disable / SYSCLK / HSE / HSI / PLL/2 | 时钟输出引脚 | **Disable** |

```text
HSE = Crystal/Ceramic Resonator：
  → 板子上有 8MHz 晶振，选这个
  → CubeMX 会自动把 PD0/PD1 配置为 OSC_IN/OSC_OUT
  → 精度高（±0.01%），适合 UART 波特率和精确定时

BYPASS：
  → 外部时钟信号直接注入（板子上有源晶振或信号发生器）
  → 普通开发板用不上

Disable：
  → 不用外部晶振，只用内部 HSI（8MHz RC 振荡器）
  → 精度差（±1%），UART 可能乱码

LSE = Disable：
  → LSE 是给 RTC（实时时钟）用的 32.768kHz 晶振
  → 平衡车不需要 RTC，关闭释放引脚
```

### Clock Configuration 页面（时钟树）

> 位置：顶部标签页 → Clock Configuration

这是 CubeMX 最复杂的页面，但理解了结构就不难：

```
时钟树从左到右读：

输入                    处理                         分发
─────                  ─────                        ─────
HSE 8MHz ──→ PLL ×9 ──→ SYSCLK 72MHz ──→ AHB /1 ──→ HCLK 72MHz (CPU)
                                              ├→ APB1 /2 ──→ 36MHz (外设)
                                              │      └→ ×2 ──→ 72MHz (Timer)
                                              └→ APB2 /1 ──→ 72MHz (外设)
                                                     └→ ADC /6 ──→ 12MHz
```

**页面操作：**

```text
1. 左边选择时钟源：
   → PLL Source Mux：选 HSE（不选 HSI）
   → 你会看到一条绿色路径亮起来

2. 中间设置 PLL：
   → PLLMUL：选 ×9（8MHz × 9 = 72MHz）
   → 如果选太高会变红（超过芯片最高频率）

3. 右边设置分频：
   → AHB Prescaler：/1（HCLK = SYSCLK）
   → APB1 Prescaler：/2（APB1 = HCLK/2 = 36MHz）
   → APB2 Prescaler：/1（APB2 = HCLK = 72MHz）
   → ADC Prescaler：/6（ADC = APB2/6 = 12MHz）

4. 颜色含义：
   → 绿色：配置正确，频率在安全范围内
   → 红色：频率超限或配置冲突
   → 灰色：未启用
```

> [!tip] APB1 Timer 倍频
> 时钟树页面上，APB1 的分频系数 /2 旁边会自动显示 Timer 时钟 = 72MHz。这是 STM32F1 的硬件特性：当 APB 分频 > 1 时，Timer 时钟自动 ×2。你在 CubeMX 界面上能看到这个值，不需要手动设置。

---

## 3. GPIO —— 引脚配置

> CubeMX 位置：System Core → GPIO → GPIO Settings 标签页

### GPIO Mode（引脚模式）

| 模式 | 含义 | 项目用例 |
|------|------|---------|
| Input mode | 通用输入 | 编码器 B 相（PB4/PB15） |
| Output mode | 通用输出 | 电机方向、STBY、LED |
| Alternate Function | 复用功能 | USART TX、I2C、TIM PWM（CubeMX 自动设） |
| Analog | 模拟模式 | ADC 输入（PB0） |
| External Interrupt Rising | 上升沿中断 | — |
| External Interrupt Falling | 下降沿中断 | — |
| **External Interrupt Rising/Falling** | **双边沿中断** | **编码器 A 相（PB3/PB14）** |

### GPIO Pull-up/Pull-down（上下拉）

| 参数 | 含义 | 选择依据 |
|------|------|---------|
| No pull-up/pull-down | 无上下拉 | 信号源自身有明确电平 |
| Pull-up | 内部上拉（默认高电平） | 按键、编码器、UART RX |
| Pull-down | 内部下拉（默认低电平） | 电机控制引脚、LED |

```text
你的项目配置逻辑：

PA11 USER_BUTTON = Pull-up
  → 按键低电平有效（按下接 GND）
  → 不按时通过上拉保持高电平，不会被噪声误触发

PB3/PB14 编码器 ENCA = Pull-up
  → 编码器输出开漏或信号不稳定时，上拉保证空闲电平确定

PB0 ADC = 无上下拉（Analog 模式自动处理）

PA1 STBY = Pull-down
  → 默认低电平 = 电机驱动休眠
  → 上电瞬间不会因为引脚浮空导致电机意外使能

PA9/PA10 电机方向 = Pull-down
  → 默认低电平 = 电机不转
  → 和 STBY 一样，上电安全第一

PA4/PA5/PA6 LED = Pull-down
  → 默认低电平 = LED 灭
```

> [!warning] 什么时候必须加外部上拉
> CubeMX 内部上拉电阻约 40kΩ，很弱。如果信号线较长、环境噪声大、或者设备是开漏输出，内部上拉可能不够，需要板子上加外部 4.7kΩ~10kΩ 上拉。I2C 总线（PB8/PB9）通常需要外部上拉。

### GPIO Speed（输出速度）

| 参数 | 含义 | 实际频率 |
|------|------|---------|
| Low | 低速 | ≤ 2MHz |
| Medium | 中速 | ≤ 10MHz |
| High | 高速 | ≤ 50MHz |

```text
速度控制的是 GPIO 输出信号的边沿陡峭程度（slew rate）：
  → 速度越高，信号翻转越快
  → 但翻转太快会产生更多电磁干扰（EMI）

你的项目：
  → PA2 USART2_TX 设为 Low：921600 baud 信号变化不快，Low 足够
  → 其他输出引脚用默认值：电机 PWM 20kHz、GPIO 翻转，Medium 就够

什么时候需要 High？
  → SPI 时钟超过 10MHz
  → 高速 PWM（几百 kHz 以上）
```

### User Label（用户标签）

```text
在 GPIO 引脚上右键 → User Label → 输入名字

作用：
  → CubeMX 在 main.h 里生成 #define 宏
  → 比如 PA1 设 Label = MOTOR_STBY
  → 生成：#define MOTOR_STBY_Pin GPIO_PIN_1
           #define MOTOR_STBY_GPIO_Port GPIOA
  → 代码里用 MOTOR_STBY_Pin 而不是 GPIO_PIN_1，可读性好

你的项目所有引脚都有 Label：
  MOTOR_STBY, MOTOR_L_IN1, MOTOR_L_IN2, MOTOR_L_PWM
  MOTOR_R_IN1, MOTOR_R_IN2, MOTOR_R_PWM
  MOTOR_L_ENCA, MOTOR_L_ENCB, MOTOR_R_ENCA, MOTOR_R_ENCB
  USER_BUTTON, LED1, LED2, LED3

⚠️ 设了 Label 后建议 Lock（引脚上右键 → Lock）
   防止后续操作误改引脚配置
```

---

## 4. EXTI —— 外部中断配置

> CubeMX 位置：在芯片引脚图上点击引脚 → 选择 GPIO_EXTIxx

### 你的项目配置

```text
PB3  → GPIO_EXTI3   → MOTOR_R_ENCA（右编码器 A 相）
PB14 → GPIO_EXTI14  → MOTOR_L_ENCA（左编码器 A 相）

在 GPIO → GPIO Settings 面板中：
  GPIO Mode: External Interrupt Mode with Rising/Falling edge trigger detection
  → 双边沿触发：上升沿和下降沿都进中断
  → 编码器 A 相的每个边沿都计数，精度翻倍
```

### EXTI 触发模式选择

| 模式 | 含义 | 适用场景 |
|------|------|---------|
| Rising edge | 只在上升沿触发 | 简单按键检测 |
| Falling edge | 只在下降沿触发 | 低电平有效的按键 |
| **Rising/Falling** | **双边沿触发** | **编码器、需要检测每个电平变化** |

```text
为什么编码器用双边沿？
  → 编码器 A 相一个完整周期 = 上升沿 + 下降沿 = 2 个中断
  → 只用上升沿：每转一圈计 N 次
  → 用双边沿：每转一圈计 2N 次，分辨率翻倍
```

---

## 5. NVIC —— 中断配置

> CubeMX 位置：System Core → NVIC → NVIC Settings 标签页

### Priority Group（优先级分组）

| 参数 | 可选项 | 含义 | 项目值 |
|------|--------|------|--------|
| Priority Group | 0~4 组 | 抢占优先级和子优先级的位数分配 | **Group 4** |

```text
STM32 中断优先级有 4 位（0~15），分成两部分：
  抢占优先级（Preemption）：数字小的可以打断数字大的
  子优先级（Sub Priority）：数字小的先响应，但不能打断

Group 4 = 4 位抢占 + 0 位子优先级
  → 16 级抢占优先级（0~15），没有子优先级
  → 0 最高，15 最低
  → 只要抢占优先级不同，高优先级可以打断低优先级

为什么选 Group 4？
  → 平衡车项目需要明确区分"谁能打断谁"
  → 不需要子优先级（同优先级的中断按自然顺序排就行）
  → Group 4 最简单直接
```

### 你项目的中断优先级分配

在 NVIC Settings 标签页中，你会看到所有已启用的中断：

| 中断 | 优先级 | 用途 | 说明 |
|------|:------:|------|------|
| EXTI3 | 0 | 右编码器 | 最高优先级，不能被任何中断打断 |
| EXTI15_10 | 0 | 左编码器 | 同上 |
| TIM2 | 0 | 微秒时间基准 | 同上 |
| USART3 | 0 | 遥控串口 | 同上 |
| ADC1_2 | 0 | 电池采样 | 同上 |
| SysTick | 15 | HAL 1ms 时钟 | 最低优先级，所有控制中断都能打断它 |

```text
为什么控制相关中断全是 0？
  → 编码器边沿必须立刻响应，否则漏脉冲
  → TIM2 时间基准必须准确，否则 dt_s 不准
  → 这些中断的 ISR 都很短（几 us），不会互相阻塞

为什么 SysTick 是 15？
  → HAL_GetTick() 只是 1ms 计数器，延迟几十 us 无所谓
  → 不能让 SysTick 打断编码器 ISR，会导致脉冲计数不准
```

### Enabled/Disabled 复选框

```text
NVIC 列表中每个中断前有个复选框：
  ☑ = 启用这个中断
  ☐ = 不启用

CubeMX 不会自动勾选所有中断，你需要手动启用：
  → 配了 EXTI 引脚 → 需要去 NVIC 页面手动勾选
  → 配了 ADC + 中断 → 需要手动勾选 ADC 中断
  → 忘了勾选 → 代码里 HAL 回调永远不会被调用
```

---

## 6. TIM —— 定时器配置（PWM 模式）

> CubeMX 位置：Peripherals → TIM1 / TIM4

### 你的项目 PWM 配置

以 TIM1（左电机 PWM）为例，Parameter Settings 页面：

| 参数 | 含义 | 项目值 | 计算结果 |
|------|------|:------:|---------|
| Prescaler (PSC) | 时钟分频 | 0 | 72MHz / (0+1) = 72MHz |
| Counter Mode | 计数方向 | Up | 从 0 数到 ARR |
| Counter Period (ARR) | 计数周期 | 3599 | 72MHz / 3600 = 20kHz |
| AutoReload Preload | ARR 预装载 | Enable | ARR 运行时修改不会立刻生效，等下一个周期 |
| Channel1 | 通道模式 | **PWM Generation CH1** | 生成 PWM 波形 |

### Channel 模式选择

| 模式 | 含义 |
|------|------|
| Disable | 不使用这个通道 |
| **PWM Generation CH1** | **PWM 输出模式 1（常用）** |
| PWM Generation CH1N | 互补 PWM 输出（高级定时器才有，用于电机驱动） |
| Output Compare CH1 | 比较输出（可以产生单脉冲、特定时序） |
| Input Capture CH1 | 输入捕获（测量脉宽、频率） |
| One Pulse CH1 | 单脉冲输出 |

```text
PWM Generation = 模式 1：
  → 计数值 < CCR 时输出有效电平
  → 计数值 ≥ CCR 时输出无效电平
  → CCR 越大，占空比越大
  → duty = CCR / (ARR + 1) × 100%

PWM 频率 = TIM_CLK / ((PSC + 1) × (ARR + 1))
  → 你的：72MHz / (1 × 3600) = 20kHz
```

### PWM Polarity（极性）

```text
在 PWM Channel 设置中：
  CH Polarity: High / Low
  
  High = 有效电平为高（duty 越大，高电平占比越大）
  Low  = 有效电平为低（duty 越大，低电平占比越大）

你的项目选 High（默认）
```

### AutoReload Preload

```text
Enable（推荐）：
  → 运行时修改 ARR，新的值在下一个计数周期开始时才生效
  → 避免在计数过程中改 ARR 导致 PWM 波形异常

Disable：
  → 改 ARR 立刻生效
  → 如果当前计数已经超过了新 ARR，可能会产生意外行为
```

### TIM4（右电机 PWM）的区别

```text
TIM1 和 TIM4 的配置完全相同：
  PSC = 0, ARR = 3599, PWM Generation CH1

区别：
  TIM1 是高级定时器 → 挂在 APB2（72MHz），有互补输出、死区插入等高级功能
  TIM4 是通用定时器 → 挂在 APB1（Timer 时钟也是 72MHz，因为 ×2 倍频）
  
  你的项目只用基础 PWM 功能，两者表现一样
```

---

## 7. TIM —— 定时器配置（基本模式 + 主从触发）

> CubeMX 位置：Peripherals → TIM2 / TIM3

### TIM2（微秒时间基准）

| 参数 | 含义 | 项目值 |
|------|------|:------:|
| Clock Source | 时钟来源 | Internal Clock |
| Prescaler (PSC) | 分频 | 71 |
| Counter Mode | 计数方向 | Up |
| Counter Period (ARR) | 计数周期 | 65535 |
| AutoReload Preload | ARR 预装载 | Enable |
| NVIC 中断 | 溢出中断 | ✑ 启用 |

```text
时钟来源选择：
  Internal Clock → 使用 APB1 Timer 时钟（72MHz）
  ETR → 外部触发输入
  ITRx → 其他定时器的输出（主从模式）

PSC = 71 → 72MHz / 72 = 1MHz（1 个计数 = 1us）
ARR = 65535 → 每 65536us（约 65.5ms）溢出一次

为什么不把 ARR 设大一点？
  → ARR 最大 65535（16 位定时器）
  → 溢出后用软件累积，65.5ms 精度对平衡车完全够用

NVIC 中断：
  → 必须启用 TIM2 全局中断
  → 否则溢出事件不会触发回调，时间会丢失
```

### TIM3（ADC 触发定时器）

| 参数 | 含义 | 项目值 |
|------|------|:------:|
| Clock Source | 时钟来源 | Internal Clock |
| Prescaler (PSC) | 分频 | 71 |
| Counter Period (ARR) | 计数周期 | 9999 |
| **Master/Slave Mode** | **主从模式** | **Enable** |
| **Trigger Event (TRGO)** | **触发输出事件** | **Update Event** |

```text
PSC = 71, ARR = 9999 → 72MHz / 72 / 10000 = 10ms 周期

TRGO（Trigger Output）是什么？
  → 定时器可以向其他外设发出"信号"
  → TRGO 选择哪个事件作为触发信号

  Update Event = 计数器溢出（从 ARR 归零）时发出触发
  → 每 10ms 触发一次 ADC 转换

Master/Slave Mode = Enable：
  → 允许这个定时器作为主设备，触发其他外设（ADC）
  → 不开启的话 TRGO 信号不会输出
```

### 主从模式图解

```
TIM3（主设备）                    ADC1（从设备）
┌──────────┐                    ┌──────────┐
│ 计数器    │                    │          │
│ 0→9999   │ ──TRGO(Update)──→ │ 触发转换   │
│ 每10ms溢出│                    │ 每10ms采样 │
└──────────┘                    └──────────┘

CubeMX 中的配置链路：
  TIM3 Parameter → TRGO = Update Event
  TIM3 Parameter → Master/Slave Mode = Enable
  ADC1 Parameter → External Trigger = TIM3 TRGO
  → 这三者配合，TIM3 自动触发 ADC，不需要 CPU 干预
```

---

## 8. ADC —— 模数转换配置

> CubeMX 位置：Peripherals → ADC1

### Parameter Settings

| 参数 | 含义 | 可选项 | 项目值 |
|------|------|--------|:------:|
| Clock Prescaler | ADC 时钟分频 | /2 /4 /6 /8 | **/6**（由 RCC 页面控制） |
| Resolution | 分辨率 | — | 12 bit（F1 固定） |
| Data Alignment | 数据对齐 | Right / Left | **Right** |
| Scan Conversion Mode | 扫描模式 | Enable / Disable | **Disable** |
| Continuous Conversion Mode | 连续转换 | Enable / Disable | **Disable** |
| Discontinuous Conversion Mode | 不连续转换 | Enable / Disable | **Disable** |
| Number of Conversion | 转换通道数 | 1~16 | **1** |

```text
Data Alignment = Right：
  → 12 位结果放在 uint16_t 的低 12 位
  → 直接读 ADC->DR 就是 0~4095
  → Left 对齐的话高位有效，需要移位处理，更麻烦

Scan Mode = Disable：
  → 只有一个通道（电池 PB0），不需要扫描
  → 如果采样多个通道（电池+温度+电位器），需要 Enable

Continuous = Disable：
  → 不连续转换，由 TIM3 触发才转换一次
  → Enable 的话 ADC 会不停地转换，费电且数据来不及处理
```

### Channel Configuration（通道配置）

在 ADC1 的 Parameter Settings 下方，有 Rank 标签：

| 参数 | 含义 | 项目值 |
|------|------|:------:|
| Channel | ADC 通道号 | Channel 8（对应 PB0） |
| Rank | 转换顺序 | 1（第一个也是唯一一个） |
| Sampling Time | 采样时间 | **239.5 Cycles** |

```text
Sampling Time 选择：

采样时间越长，ADC 内部电容充电越充分，结果越准确
但转换也越慢

239.5 cycles 的含义：
  → ADC 采样电容充电 239.5 个 ADC 时钟周期
  → 加上 12.5 cycles 的转换时间
  → 总转换 = (239.5 + 12.5) / 12MHz ≈ 21us
  → TIM3 每 10ms 触发一次，21us 远小于 10ms，没问题

为什么选最大的？
  → 电池电压采样对速度没要求
  → 采样时间越长越稳定，抗噪声越好
  → 如果采样高速信号（音频），需要选更短的采样时间
```

### External Trigger（外部触发）

| 参数 | 含义 | 项目值 |
|------|------|:------:|
| External Trigger Conversion Source | 触发源 | **Tim 3 Trigger Out event** |
| External Trigger Conversion Edge | 触发边沿 | **Rising Edge** |

```text
触发源选 TIM3 TRGO：
  → TIM3 每 10ms 产生一个 TRGO 信号
  → ADC 收到信号后自动开始转换
  → 不需要 CPU 调用 HAL_ADC_Start()

触发边沿选 Rising Edge：
  → TRGO 信号的上升沿触发转换
  → 下降沿、双边沿也可以选，但 Rising 最常用
```

---

## 9. I2C —— 通信配置

> CubeMX 位置：Peripherals → I2C1

### Parameter Settings

| 参数 | 含义 | 可选项 | 项目值 |
|------|------|--------|:------:|
| I2C Speed Mode | 速度模式 | Standard / Fast | **Standard** |
| I2C Clock Speed | 时钟频率 | 0~400kHz | **100kHz** |
| Clock No Stretch Mode | 时钟拉伸 | Enable / Disable | Disable |

```text
Standard Mode = 100kHz：
  → MPU6050 支持 400kHz（Fast Mode）
  → 但 100kHz 对 200Hz 采样率绰绰有余
  → Standard Mode 信号裕量更大，更稳定
  → 如果你发现 I2C 通信不稳定，可以先降速排查

I2C Clock Speed = 100000：
  → 直接输入 100000（100kHz）
  → CubeMX 会自动计算 CCR 寄存器值
  → 你不需要手动算时序参数

引脚：
  PB8 = SCL（时钟线）
  PB9 = SDA（数据线）
  → 这是 I2C1 的重映射引脚
  → 如果你选错引脚，I2C 不会工作
```

### I2C 模式选择

| 模式 | 速度 | 适用场景 |
|------|------|---------|
| Standard | 100kHz | 大多数传感器，稳定可靠 |
| Fast | 400kHz | MPU6050 也支持，适合高采样率 |
| Fast Plus | 1MHz | 需要特定 MCU 支持 |
| High Speed | 3.4MHz | 极少用 |

---

## 10. USART —— 串口配置

> CubeMX 位置：Peripherals → USART2 / USART3

### USART 模式认知

![](assets/STM32CubeMX/file-20260512090035756.png)

### 最核心的模式

- **Asynchronous (异步通信)**
  - 含义：普通串口 UART，不需要时钟线，TX/RX 两根线 + GND，约定波特率通信。
  - 应用场景：printf 调试、蓝牙模块、WiFi 模块、GPS、CH340 连电脑，**全部选这个**。

### 其他模式（了解即可）

| 模式 | 含义 | 说明 |
|------|------|------|
| Disable | 关闭外设 | 释放引脚，省电 |
| Synchronous | 同步通信，多一根 CK 时钟线 | 很少用，一般直接用 SPI/I2C |
| Single Wire | 单线半双工 | 引脚紧张时用，需要代码控制收发切换 |
| Multiprocessor | 多机通信 | 利用第 9 位区分地址/数据，现在多用 RS-485/CAN |
| IrDA | 红外通信 | 红外遥控，已被蓝牙取代 |
| LIN | 低成本车载总线 | 汽车电子专用 |
| SmartCard | 智能卡 ISO 7816 | POS 机、读卡器 |

### Parameter Settings

以 USART2（调试串口）为例：

| 参数 | 含义 | USART2 值 | USART3 值 |
|------|------|:---------:|:---------:|
| Baud Rate | 波特率 | **921600** | **9600** |
| Word Length | 数据位 | 8 Bits | 8 Bits |
| Parity | 校验位 | None | None |
| Stop Bits | 停止位 | 1 | 1 |
| Data Direction | 数据方向 | Receive and Transmit | Receive and Transmit |
| Over Sampling | 过采样 | 16 Samples | 16 Samples |

```text
Baud Rate = 921600（USART2）：
  → 调试串口用高速，打印大量数据不阻塞
  → 921600 是 72MHz 时钟下常见的最高可靠波特率

Baud Rate = 9600（USART3）：
  → 遥控串口匹配蓝牙模块或串口助手的默认波特率
  → 9600 虽然慢，但遥控命令很短（"move 0 50\n"），够用

波特率是怎么算出来的？
  → BRR = PCLK / BaudRate
  → USART2 挂在 APB1：36MHz / 921600 ≈ 39.06 → 取整 39
  → 实际波特率 = 36MHz / 39 ≈ 923077，误差约 0.16%，可以接受
  → CubeMX 在界面上会显示 Error(%): 0.xx%
```

### 引脚上下拉

```text
USART2 TX (PA2)：默认配置即可（复用推挽输出，CubeMX 自动处理）
USART2 RX (PA3)：设 Pull-up
  → RX 空闲时应保持高电平（UART 空闲 = 高）
  → 不接设备时不至于因为浮空产生噪声数据

USART3 RX (PB11)：同上，设 Pull-up
```

---

## 11. Project Manager —— 工程配置

> CubeMX 位置：顶部标签页 → Project Manager

### Project 设置

| 参数 | 含义 | 项目值 |
|------|------|:------:|
| Project Name | 工程名 | HAL-Car |
| Toolchain / IDE | 工具链 | **MDK-ARM V5** |
| Firmware Package | HAL 库版本 | STM32Cube FW_F1 V1.8.7 |

### Code Generator 设置

| 参数 | 含义 | 项目值 | 说明 |
|------|------|:------:|------|
| Generate peripheral initialization as a pair of .c/.h | 每个外设生成独立文件 | ☑ | 不勾选的话所有初始化塞进 main.c |
| Keep User Code when re-generating | 重新生成时保留用户代码 | ☑ | **必须勾选** |
| Set all free pins as analog | 未用引脚设为模拟模式 | ☐ | 省电优化，非必须 |
| Full Assert | HAL 断言 | ☐ | 调试用，Release 关闭 |

### Linker Settings（链接器设置）

| 参数 | 含义 | 项目值 | 说明 |
|------|------|:------:|------|
| Stack Size | 栈大小 | **0x400 (1KB)** | 局部变量、函数调用、中断上下文 |
| Heap Size | 堆大小 | **0x200 (512B)** | malloc 动态分配（HAL 可能用到） |

```text
Stack Size 选择：
  → 0x400 = 1024 字节
  → 你需要考虑最深的函数调用链 + 中断嵌套的栈消耗
  → 你的最深链路：ISR → HAL 回调 → bsp_encoder_exti → 约 100 字节
  → 加上主循环的 app_state_update → balance_controller → PID → 约 200 字节
  → 1KB 留有余量，但对于 printf（栈消耗大）可能偏紧
  → 如果运行时出现 HardFault，可以尝试增大到 0x800

Heap Size 选择：
  → 0x200 = 512 字节
  → 你的项目没有用 malloc，这个值够用
  → HAL 某些函数可能内部调 malloc，但不多
```

> [!warning] 栈溢出是最难调试的 Bug
> 栈溢出不会报错，只会产生 HardFault 或数据莫名其妙被篡改。如果你发现全局变量的值莫名其妙变了，或者进入 HardFault_Handler，首先怀疑栈溢出，把 Stack Size 加大一倍试试。

---

## 附：你的项目完整引脚分配一览

| 引脚 | 功能 | CubeMX 配置 | User Label |
|------|------|------------|-----------|
| PA1 | 电机 STBY | GPIO_Output, Pull-down | MOTOR_STBY |
| PA2 | 调试串口 TX | USART2_TX | — |
| PA3 | 调试串口 RX | USART2_RX, Pull-up | — |
| PA4 | LED1 | GPIO_Output, Pull-down | LED1 |
| PA5 | LED2 | GPIO_Output, Pull-down | LED2 |
| PA6 | LED3 | GPIO_Output, Pull-down | LED3 |
| PA8 | 左电机 PWM | TIM1_CH1 | MOTOR_L_PWM |
| PA9 | 左电机 IN1 | GPIO_Output, Pull-down | MOTOR_L_IN1 |
| PA10 | 左电机 IN2 | GPIO_Output, Pull-down | MOTOR_L_IN2 |
| PA11 | 用户按键 | GPIO_Input, Pull-up | USER_BUTTON |
| PA13 | SWD 调试 | SYS_JTMS-SWDIO | — |
| PA14 | SWD 调试 | SYS_JTCK-SWCLK | — |
| PB0 | 电池 ADC | ADC1_IN8 | — |
| PB3 | 右编码器 A | GPIO_EXTI3, Both edge, Pull-up | MOTOR_R_ENCA |
| PB4 | 右编码器 B | GPIO_Input, Pull-up | MOTOR_R_ENCB |
| PB5 | 右电机 IN1 | GPIO_Output, Pull-down | MOTOR_R_IN1 |
| PB6 | 右电机 PWM | TIM4_CH1 | MOTOR_R_PWM |
| PB7 | 右电机 IN2 | GPIO_Output, Pull-down | MOTOR_R_IN2 |
| PB8 | I2C 时钟 | I2C1_SCL | — |
| PB9 | I2C 数据 | I2C1_SDA | — |
| PB10 | 遥控串口 TX | USART3_TX | — |
| PB11 | 遥控串口 RX | USART3_RX, Pull-up | — |
| PB14 | 左编码器 A | GPIO_EXTI14, Both edge, Pull-up | MOTOR_L_ENCA |
| PB15 | 左编码器 B | GPIO_Input, Pull-up | MOTOR_L_ENCB |
| PD0 | 外部晶振 | RCC_OSC_IN | — |
| PD1 | 外部晶振 | RCC_OSC_OUT | — |

---

## CubeMX 常见操作速查

| 操作 | 方法 |
|------|------|
| 选芯片 | File → New Project → 搜索 STM32F103C8 |
| 配引脚功能 | 在芯片图上点击引脚 → 选择功能 |
| 改引脚参数 | 左侧点外设 → 下方 Parameter Settings |
| 配时钟树 | 顶部 Clock Configuration 标签页 |
| 生成代码 | 顶部 Project Manager → Generate Code 按钮 |
| 重新生成 | 改完参数后直接点 Generate Code，已有 USER CODE 不会丢失 |
| 锁定引脚 | 引脚上右键 → Lock，防止误改 |
| 查看引脚冲突 | 引脚变红 = 冲突，需要修改功能分配 |
| 导出 PDF 引脚图 | Tool → Pinout Export to PDF |

---

## 后续连接

- [[工程经验]]：每个模块在代码层面的设计决策
- [[文件架构思路]]：CubeMX 生成的代码和用户代码的边界
- [[面试提问]]：面试中关于 CubeMX 和外设配置的常见问题
