---
tags:
  - 嵌入式
  - 中断
  - RT-Thread
  - RTOS
aliases:
  - RT-Thread中断管理
  - rt_interrupt
  - RT-Thread ISR
  - rt_hw_context_switch
related:
  - "[[中断在RTOS中的角色]]"
  - "[[PendSV]]"
  - "[[栈与上下文]]"
  - "[[中断的基础理解]]"
  - "[[向量表的基础理解]]"
  - "[[../操作系统与内核/04_FreeRTOS/内存管理/中断推迟处理]]"
  - "[[../源码阅读/RT-thread源码阅读-v2/05-中断-Tick-定时器]]"
chip: STM32
---

# RT-Thread 中断机制

> [!abstract]
> RT-Thread 把中断处理分为三段：前导（保存上下文 + nesting++）、用户 ISR（业务逻辑）、后续（nesting-- + 调度决策 + 恢复上下文）。调度通过 `rt_hw_context_switch_interrupt` 设置 flag 并挂起 PendSV 延迟执行。中断栈独立于线程栈，节省内存。高速场景下应考虑轮询替代中断。

> [!info] 面试开场句
> "RT-Thread 中断分三段：前导保存上下文并递增嵌套计数，用户 ISR 执行业务逻辑，后续递减嵌套计数并决定是否调度。调度不是在 ISR 里直接做的，而是通过 rt_hw_context_switch_interrupt 设置 flag 并挂起 PendSV，等所有中断退出后才真正切换线程。"

---

## 一、中断处理的三段式结构

### 1.1 总览

```
硬件中断信号来了
    │
    ▼
┌─────────────────────────────────────────┐
│ ① 中断前导（Interrupt Preamble）         │
│   a. 硬件自动保存上下文（Cortex-M 压 8 个）│
│   b. rt_interrupt_enter() → nesting++    │
│   c. 调用用户 ISR                        │
├─────────────────────────────────────────┤
│ ② 用户 ISR（User ISR）                   │
│   清标志位、读数据、调 rt_event_send()    │
│   可能触发 rt_hw_context_switch_interrupt │
├─────────────────────────────────────────┤
│ ③ 中断后续（Interrupt Follow-up）        │
│   a. rt_interrupt_leave() → nesting--    │
│   b. 恢复上下文                           │
│      - 无线程切换 → 恢复原来线程           │
│      - 有线程切换 → PendSV 切到新线程      │
└─────────────────────────────────────────┘
```

### 1.2 完整时间线

```
线程X 正在执行（用 PSP）
  │
  ├── 硬件中断信号
  │
  ├── 前导阶段：
  │   ① CPU 硬件自动压栈 8 个寄存器（到 PSP 指向的线程栈）
  │   ② CPU 切换到 MSP
  │   ③ rt_interrupt_enter() → nesting++ （全局变量 +1）
  │   ④ 调用用户注册的 ISR 函数
  │
  ├── 用户 ISR 阶段：
  │   ⑤ 清中断标志位
  │   ⑥ 读数据 / 写缓冲区
  │   ⑦ 调 rt_event_send() / rt_sem_release() 等 IPC
  │      → 内部检查 nesting > 0 → 只记账，不立刻调度
  │   ⑧ 如果需要切换线程 → 调 rt_hw_context_switch_interrupt()
  │      → 设置 flag，挂起 PendSV
  │
  ├── 后续阶段：
  │   ⑨ rt_interrupt_leave() → nesting--
  │   ⑩ 检查 nesting == 0 ?
  │       → 否：恢复上一层中断的上下文（嵌套返回）
  │       → 是：检查是否需要调度
  │           → 不需要：CPU 切回 PSP，恢复线程X
  │           → 需要：PendSV 执行，切换到新线程
```

### 1.3 两种结果

| 情况 | ISR 行为 | 后续处理 |
|------|---------|---------|
| **不切换线程** | ISR 中没有触发调度 | nesting==0 → 恢复线程X → 继续 |
| **切换线程** | ISR 中调了 IPC 唤醒更高优先级线程 | nesting==0 → PendSV → 切到新线程 |

---

## 二、rt_hw_context_switch_interrupt 源码解析

### 2.1 三个全局变量

```c
rt_ubase_t rt_interrupt_from_thread = 0;       // 被切走线程的 TCB 地址
rt_ubase_t rt_interrupt_to_thread = 0;         // 目标线程的 TCB 地址
volatile rt_ubase_t rt_thread_switch_interrupt_flag = 0;  // 是否有待处理的切换
```

### 2.2 源码逐行解析

```c
void rt_hw_context_switch_interrupt(rt_ubase_t from, rt_ubase_t to) {
    if (rt_thread_switch_interrupt_flag != 0) {
        // 已经有一次调度在排队，只更新目标线程
        rt_interrupt_to_thread = to;
        return;
    }

    rt_interrupt_from_thread = from;            // 记录旧线程
    rt_interrupt_to_thread = to;                // 记录新线程
    rt_thread_switch_interrupt_flag = 1;        // 标记需要切换

    // 挂起 PendSV（不会立刻执行，优先级最低）
    SCB->ICSR = SCB_ICSR_PENDSVSET_Msk;
}
```

### 2.3 flag 重复判断的原因

```
场景：SysTick ISR 和 UART ISR 都想触发调度

时刻1：SysTick ISR 执行
  → rt_hw_context_switch_interrupt(from, thread_A)
  → flag = 1, to_thread = A, PendSV 挂起

时刻2：SysTick 还没退出，UART ISR 嵌套进来
  → rt_hw_context_switch_interrupt(from, thread_B)
  → flag 已经是 1！
  → 只更新 to_thread = B
  → 不重复挂起 PendSV（已经挂起了，没必要再挂）

时刻3：两个 ISR 都退出 → PendSV 执行
  → PendSV 内部会调 rt_schedule() 重新找最高优先级任务
  → 所以只需更新 to_thread，调度器会自动纠正
```

> [!tip] PendSV 只需要执行一次。它执行时会调用 rt_schedule()，调度器自动找当前最高优先级任务。多次挂起没有意义。

---

## 三、中断栈 —— 独立 vs 共享

### 3.1 两种方案

```
方案A：中断栈和线程栈共用（Cortex-M 裸机默认）
  → 中断压栈压到当前线程的 PSP 上
  → 每个线程都必须预留中断栈空间

方案B：中断栈独立（RTOS 的做法）
  → 中断使用 MSP（独立系统栈）
  → 线程使用各自的 PSP
  → 中断嵌套只消耗 MSP 空间
```

### 3.2 不分开的灾难

```
假设不分开：中断来了，ISR 的局部变量压入当前线程的栈
  → 支持中断嵌套时，压栈深度不可控
  → 每个线程创建时必须多分配 1~2KB 防备中断
  → 50 个线程 = 浪费 50~100KB 宝贵 RAM
```

### 3.3 分开的精妙

```
独立中断栈（MSP）：
  → 所有中断共用一个 MSP
  → 无论嵌套多深，只消耗这一个栈
  → 线程栈只需考虑自身函数调用深度
  → 线程越多，节省的内存越明显
```

| 方案 | 线程栈需要预留中断空间 | 内存利用率 |
|------|---------------------|-----------|
| 共享 | 每个线程 +1~2KB | 低（线程越多越浪费） |
| 独立 | 不需要 | 高（所有中断共用一个 MSP） |

> [!important] 这就是 RT-Thread 用 MSP/PSP 双栈的根本原因。详见 [[栈与上下文#二、两个栈指针：MSP 与 PSP]]。

---

## 四、中断管理接口

### 4.1 中断源管理

```c
// 屏蔽指定中断源（不响应）
void rt_hw_interrupt_mask(int vector);

// 解除屏蔽
void rt_hw_interrupt_umask(int vector);
```

典型用法：

```c
// 配置 UART 中断前先屏蔽
rt_hw_interrupt_mask(USART1_IRQn);

// 配置 UART 外设、NVIC、ISR 注册...

// 配置完成后解除屏蔽
rt_hw_interrupt_umask(USART1_IRQn);
```

### 4.2 全局中断开关 —— 状态保存机制

```c
// 关中断：返回当前中断状态（不是简单的 0/1）
rt_base_t rt_hw_interrupt_disable(void);

// 恢复到之前的状态（不是"开中断"！）
void rt_hw_interrupt_enable(rt_base_t level);
```

#### 套娃原理

```
场景：函数 A 调用函数 B，两层都关中断

level_A = rt_hw_interrupt_disable();   // 关中断，level_A = 之前状态（开）
  // A 的临界区开始
  level_B = rt_hw_interrupt_disable(); // 关中断，level_B = 之前状态（关）
    // B 的临界区
  rt_hw_interrupt_enable(level_B);     // 恢复到 level_B = 关 → 中断还是关的 ✅
  // A 继续，中断仍然关闭 ✅
rt_hw_interrupt_enable(level_A);       // 恢复到 level_A = 开 → 中断开了 ✅
```

#### 不用状态保存的灾难

```
假设用简单的 disable / enable：
  disable();    // A 关中断
    disable();  // B 关中断（多余，已经关了）
    enable();   // B 开中断！！！A 的保护被破坏了！
  // A 以为自己还在保护中，实际中断已开 → 数据竞争
```

> [!tip] rt_hw_interrupt_enable 不是"开中断"，而是"恢复到之前的状态"。这是嵌套安全的关键。好比连环套娃，每次解开一层只恢复上一层状态，只有解开最外层时中断才真正打开。

### 4.3 中断通知

```c
void rt_interrupt_enter(void);   // 进入中断时调用，nesting++
void rt_interrupt_leave(void);   // 退出中断时调用，nesting--
```

作用：让内核知道当前是否在中断中。

```c
// 内部使用示例（rt_sem_release 等 IPC 函数中）
if (rt_interrupt_get_nest() != 0) {
    // 在中断中 → 只做记账，不立刻调度
} else {
    // 在线程中 → 可以立刻调度
}
```

### 4.4 关中断 vs 临界区 vs 互斥量

| | 关中断 | 临界区 | 互斥量 |
|--|-------|--------|--------|
| 实现级别 | 硬件级（CPSID I） | 软件级（锁调度器） | 软件级（锁资源） |
| 保护范围 | 整个 CPU 不响应中断 | 不做任务切换 | 只阻塞访问同一资源的线程 |
| 中断是否响应 | **不响应** | 正常响应 | 正常响应 |
| 调度是否触发 | 不触发 | 不触发 | 可以触发 |
| 性能影响 | 最大 | 中等 | 最小 |
| 适用场景 | ISR 与线程共享数据 | 线程间短临界区 | 线程间共享资源 |
| API | `rt_hw_interrupt_disable/enable` | `rt_enter_critical/exit_critical` | `rt_mutex_take/release` |

```
类比：
  关中断  → 拉断整栋楼的电闸（硬件级，谁都别想用电）
  临界区  → 反锁办公室的门（软件级，别人进不来，但楼里其他活动正常）
  互斥量  → 给共享文件柜上锁（只锁这一个资源，不影响其他工作）
```

> [!warning] **关键区别**：临界区不能保护 ISR 和线程之间的共享数据！临界区只锁调度器，中断仍然正常响应。ISR 是硬件级的，不管调度器锁没锁。ISR 和线程共享数据必须用关中断保护。

---

## 五、上半部/下半部在 RT-Thread 中的实现

### 5.1 生产者-消费者模型

```
ISR（暴躁的生产者）：
  只管粗暴地把数据扔进 Buffer 并敲一下锣（rt_sem_release / rt_event_send）
  快进快出，绝不停留

线程（沉稳的消费者）：
  平时睡觉（阻塞在 rt_event_recv / rt_mb_recv）
  听到锣声就醒来慢条斯理地处理数据
  可以做任何耗时操作：printf、浮点运算、调阻塞 API
```

### 5.2 IPC 机制选择

| IPC 机制 | ISR 中调用 | 传递内容 | 适用场景 |
|----------|-----------|---------|---------|
| 事件集 | `rt_event_send()` | 事件标志位 | "发生了什么事"（纯通知） |
| 信号量 | `rt_sem_release()` | 无数据，只计数 | "来了几个事件"（计数） |
| 邮箱 | `rt_mb_send()` | 4 字节（指针或值） | "传一个数据地址" |
| 消息队列 | `rt_mq_send()` | 任意长度数据 | "传一块数据" |

### 5.3 完整代码示例

```c
#include <rtthread.h>

static rt_event_t sensor_event;
static rt_mailbox_t sensor_mb;
static rt_thread_t sensor_thread;

#define SENSOR_DATA_EVT  (1 << 0)

void sensor_thread_entry(void *param) {
    rt_uint32_t evt;
    uint8_t *data;

    while (1) {
        // 阻塞等待事件（释放 CPU）
        rt_event_recv(sensor_event, SENSOR_DATA_EVT,
                      RT_EVENT_FLAG_AND | RT_EVENT_FLAG_CLEAR,
                      RT_WAITING_FOREVER, &evt);

        // 从邮箱取数据并处理
        while (rt_mb_recv(sensor_mb, (rt_ubase_t *)&data, RT_NO_WAIT) == RT_EOK) {
            process_sensor_data(data);
            rt_free(data);
        }
    }
}

void EXTI15_10_IRQHandler(void) {
    if (EXTI_GetITStatus(EXTI_Line13) != RESET) {
        EXTI_ClearITPendingBit(EXTI_Line13);

        uint8_t *buf = rt_malloc(sizeof(uint8_t));
        if (buf) {
            *buf = read_sensor();
            if (rt_mb_send(sensor_mb, (rt_ubase_t)buf) != RT_EOK) {
                rt_free(buf);
            }
        }

        rt_event_send(sensor_event, SENSOR_DATA_EVT);
    }
}

int sensor_app_init(void) {
    sensor_event = rt_event_create("sen_evt", RT_IPC_FLAG_PRIO);
    sensor_mb = rt_mb_create("sen_mb", 32, RT_IPC_FLAG_PRIO);

    sensor_thread = rt_thread_create("sensor",
                                      sensor_thread_entry,
                                      RT_NULL, 1024, 5, 20);
    rt_thread_startup(sensor_thread);
    return 0;
}
```

---

## 六、中断 vs 轮询 —— 何时该换思路

### 6.1 低速设备的蜜月期：中断是王

```
比喻：烧水需要 10 分钟
  → 定个闹钟（设中断），去看电视（执行其他线程）
  → 闹钟响了去倒水（ISR 处理）
  → 高效，互不耽误
```

### 6.2 高速设备的灾难：中断开销

```
场景：10Mbps 高速串口，每次发 32 字节

算一笔时间账：
  发送 32 字节耗时：25 微秒
  → 每隔 25 微秒来一个中断

中断的"手续费"（上下文切换）：
  保存寄存器 + 跳转 + 查表 + 恢复：约 8 微秒

经济账：
  有效工作时间：25 / (25 + 8) = 75.8%
  24% 的 CPU 时间浪费在切换上！

如果用了下半部机制（信号量唤醒线程）：
  切换开销更大 → CPU 沦为疯狂切换线程的奴隶
```

### 6.3 三种出路

| 方案 | 做法 | 优势 | 适用场景 |
|------|------|------|---------|
| **轮询** | 低优先级线程 `while(1)` 读数据 | 带宽利用率 100% | 极高速、持续数据流 |
| **DMA + 批量搬运** | 单次搬 1024 字节再中断 | 手续费平摊 | 高速但可以攒批 |
| **降低轮询优先级** | 轮询线程优先级仅高于 Idle | 兼顾吞吐和实时性 | 高速 + 有实时任务 |

### 6.4 DMA 的硬件独立性

```
轮询线程被高优先级线程抢占时：
  → CPU 切到高优先级线程
  → DMA 控制器独立运行，继续搬运数据到内存缓冲区
  → DMA 根本不知道 CPU 切了线程
  → 轮询线程恢复后，DMA 可能已经搬完一批数据
  → 直接处理已就绪的数据

DMA 不需要 CPU 参与，被抢占也照常工作。
这是高速场景推荐 DMA + 低优先级轮询线程的根本原因。
```

> [!important] 中断不是银弹。当外设速度逼近 RTOS 调度极限时，轮询反而更优。详见 [[中断在RTOS中的角色#九、数据风暴 —— 当中断来得太快]]。

---

## 七、关键概念速查

| 概念 | 说明 |
|------|------|
| 三段式结构 | 前导（保存上下文 + nesting++）→ ISR → 后续（nesting-- + 调度） |
| rt_hw_context_switch_interrupt | 设置切换 flag + 挂起 PendSV，延迟调度 |
| 嵌套计数 nesting | > 0 时不调度，== 0 时才允许切换线程 |
| 独立中断栈 | MSP 给中断用，PSP 给线程用，节省内存 |
| 状态保存机制 | 关中断时保存状态，开中断时恢复状态，支持嵌套 |
| 临界区 | 锁调度器不锁中断，不能保护 ISR 和线程的共享数据 |
| 上半部/下半部 | ISR 只做最少工作，重活交给线程（生产者-消费者模型） |
| 轮询替代中断 | 高速场景下中断开销占比过大，改用轮询或 DMA |

---

## 八、面试高频问题

> [!example]- Q1：RT-Thread 中断处理的三段式结构？
> 前导：硬件保存上下文 + rt_interrupt_enter()（nesting++）。用户 ISR：执行业务逻辑，可能触发 rt_hw_context_switch_interrupt 设置调度 flag。后续：rt_interrupt_leave()（nesting--），nesting==0 时检查是否需要调度，需要则 PendSV 切换线程。

> [!example]- Q2：rt_hw_context_switch_interrupt 的 flag 为什么有重复判断？
> 中断嵌套时可能有多个 ISR 都想触发调度。flag 已经是 1 说明 PendSV 已经挂起了，只需更新 to_thread 为最新目标。PendSV 执行时会调 rt_schedule() 重新找最高优先级任务，所以多次挂起没有意义。

> [!example]- Q3：关中断 vs 临界区 vs 互斥量的区别？
> 关中断是硬件级，禁止所有中断响应，整个 CPU 不响应外部事件。临界区是软件级，只锁调度器不做任务切换，但中断仍然正常响应。互斥量最优雅，只阻塞访问同一资源的线程。关键区别：临界区不能保护 ISR 和线程的共享数据，因为中断不管调度器锁没锁。

> [!example]- Q4：为什么临界区不能保护 ISR 和线程的共享数据？
> 临界区只锁调度器（禁止任务切换），但中断仍然正常响应。ISR 是硬件级打断，不管调度器的状态。线程在临界区里操作共享变量时，ISR 仍然可以打断并修改同一个变量，造成数据竞争。ISR 和线程共享数据必须用关中断保护。

> [!example]- Q5：什么时候该用轮询替代中断？
> 当外设速度极快，中断触发频率接近 RTOS 调度极限时。比如 10Mbps 串口每 25 微秒来一个中断，上下文切换开销 8 微秒，24% CPU 时间浪费在切换上。此时应改用低优先级轮询线程或 DMA + 批量搬运。

> [!example]- Q6：独立中断栈的好处？
> 所有中断共用一个 MSP，无论嵌套多深只消耗这一个栈空间。线程栈不需要预留中断空间，只需考虑自身函数调用深度。线程越多，节省的内存越明显（避免每个线程都多分配 1~2KB）。

---

## 九、踩坑记录

> [!bug] 实战经验填充区
> （项目开发中遇到的 RT-Thread 中断相关问题记录于此）
