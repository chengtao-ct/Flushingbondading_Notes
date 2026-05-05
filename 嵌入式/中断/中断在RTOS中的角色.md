---
tags:
  - 嵌入式
  - 中断
  - RTOS
  - RT-Thread
  - FreeRTOS
aliases:
  - ISR与任务通信
  - 中断与RTOS
  - FromISR
  - 中断延迟调度
  - ISR上下文
related:
  - "[[中断的基础理解]]"
  - "[[栈与上下文]]"
  - "[[PendSV]]"
  - "[[../操作系统与内核/04_FreeRTOS/内存管理/中断推迟处理]]"
  - "[[../源码阅读/RT-thread源码阅读-v2/05-中断-Tick-定时器]]"
chip: STM32
---

# 中断在 RTOS 中的角色

> [!abstract]
> 裸机中断只需设 flag，RTOS 中断必须"传数据 + 唤醒任务 + 触发调度"。ISR 没有任务上下文，不能阻塞、不能调普通 API，必须用 FromISR 版本。延迟调度（先记账后结账）是 RTOS 中断管理的核心设计思想。

> [!info] 面试开场句
> "RTOS 中断的核心问题是 ISR 没有任务上下文。裸机靠轮询 flag，RTOS 靠 IPC 主动通知。ISR 中只能用 FromISR 版 API，绝不阻塞，只做记账；中断退出后由调度器统一结账，决定是否切换任务。"

---

## 一、裸机中断 vs RTOS 中断 —— 根本区别

### 1.1 协作模型的不同

```
裸机：
  main() 是唯一的执行流，永远在跑 while(1) 轮询
  → ISR 设 flag，main 下一次循环就能看到
  → 不需要"通知"机制，因为 main 一直在"看"

RTOS：
  多个任务并发执行，每个任务可能在不同状态：
    任务A 在 rt_event_recv() 里阻塞（等着被唤醒）
    任务B 在 rt_thread_delay(1000) 里睡眠（1秒后才醒）
    任务C 在 rt_mb_recv() 里等数据
    任务D 正在执行

  ISR 设一个普通 flag：
    → 任务A 在阻塞，看不到
    → 任务B 在睡眠，看不到
    → 任务C 在等数据，看不到
    → 只有任务D 下次碰巧读到才行 → 靠运气

  ISR 调 rt_event_send()：
    → 直接把任务A 从阻塞态移到就绪态
    → 任务A 下次被调度执行
    → 不靠轮询，不靠运气，靠"主动通知"
```

> [!important] 本质区别：裸机靠"轮询"（主循环一直在看），RTOS 靠"通知"（ISR 主动唤醒任务）。

### 1.2 中断不再是孤立的

```
裸机：
  中断 → ISR 处理 → 回到 main
  没有任务、没有调度、没有优先级抢占

RTOS：
  中断 → ISR 处理 → 可能唤醒更高优先级任务 → 需要调度 → 可能切换任务
  中断不再是"处理完就回去"，而是"处理完可能引发任务切换"
```

**核心矛盾：ISR 没有任务上下文，但它会影响任务调度。**

---

## 二、中断嵌套计数

RTOS 用全局计数器跟踪当前是否在中断中：

### 2.1 RT-Thread

```c
volatile uint8_t rt_interrupt_nest = 0;

void rt_interrupt_enter(void) {
    rt_interrupt_nest++;
}

void rt_interrupt_leave(void) {
    rt_interrupt_nest--;
    if (rt_interrupt_nest == 0) {
        rt_schedule();  // 所有中断退出后检查是否需要调度
    }
}
```

### 2.2 FreeRTOS

```c
volatile uint32_t ulInterruptNesting = 0;

void vPortEnterInterrupt(void) {
    ulInterruptNesting++;
}

void vPortExitInterrupt(void) {
    ulInterruptNesting--;
    if (ulInterruptNesting == 0) {
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

### 2.3 铁律

```
nesting > 0  →  不调度，回到被中断的代码继续执行
nesting == 0 →  允许调度，检查是否有更高优先级任务就绪
```

---

## 三、FromISR —— 为什么中断里要用特殊 API

### 3.1 错误写法

```c
// ❌ ISR 中使用普通 API
void UART_IRQHandler(void) {
    xQueueSend(xQueue, &data, portMAX_DELAY);  // 会阻塞！系统崩溃！
}
```

为什么错：

```
如果队列满了：
  → xQueueSend 把当前任务加入等待队列
  → 调用 vTaskDelay → 触发任务切换
  → 但 ISR 不是任务，没有 TCB，没有任务上下文
  → 系统直接崩溃 / 死锁
```

### 3.2 正确写法

```c
// ✅ ISR 中使用 FromISR 版本
void UART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);
    // 不阻塞！队列满了直接返回 err
    // 如果唤醒了更高优先级任务，woken = pdTRUE
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    // 中断退出后检查是否需要调度
}
```

### 3.3 普通 API vs FromISR 对比

| | 普通 API | FromISR 版本 |
|--|---------|-------------|
| 阻塞 | 可以阻塞（等待信号/队列） | **绝不阻塞**，失败直接返回错误 |
| 调度 | 内部直接触发调度 | 只设 flag，延迟调度 |
| 上下文 | 运行在任务上下文 | 运行在中断上下文 |
| 超时 | 可以指定 timeout | 没有 timeout 参数 |
| FreeRTOS | `xQueueSend`、`xSemaphoreGive` | `xQueueSendFromISR`、`xSemaphoreGiveFromISR` |
| RT-Thread | `rt_sem_release`（统一API，内部判断） | 同左（自动适配） |

> [!warning] FreeRTOS 的铁律：ISR 中调用的任何 RTOS API 都必须是 FromISR 版本。违反必崩。

---

## 四、ISR 与任务的通信机制

ISR 里调用的 IPC 函数，同时做了**三件事**：

```
① 传递数据/信号（裸机 flag 的升级版）
② 唤醒等待中的任务（改变任务状态：阻塞 → 就绪）
③ 标记需要调度（可能触发 PendSV）
```

### 4.1 RT-Thread 的 IPC 机制在中断中的使用

| IPC 机制 | ISR 中调用的函数 | 传递内容 | 适用场景 |
|----------|----------------|---------|---------|
| 事件集 | `rt_event_send()` | 事件标志位（位掩码） | "发生了什么事"（信号通知） |
| 信号量 | `rt_sem_release()` | 无数据，只计数 | "来了几个事件"（计数） |
| 邮箱 | `rt_mb_send()` | 4 字节（指针或值） | "传一个数据地址" |
| 消息队列 | `rt_mq_send()` | 任意长度数据 | "传一块数据" |

### 4.2 RT-Thread 完整示例

```c
#include <rtthread.h>

static rt_event_t uart_event;
static rt_mailbox_t uart_mb;
static rt_thread_t uart_rx_thread;

#define UART_RX_EVT  (1 << 0)

// ===== 下半部：处理线程 =====
void uart_rx_thread_entry(void *param) {
    rt_uint32_t evt;
    rt_err_t ret;
    uint8_t *data;
    
    while (1) {
        // 阻塞等待事件（释放 CPU，别的线程可以跑）
        ret = rt_event_recv(uart_event, UART_RX_EVT,
                            RT_EVENT_FLAG_AND | RT_EVENT_FLAG_CLEAR,
                            RT_WAITING_FOREVER, &evt);
        
        if (ret == RT_EOK) {
            while (rt_mb_recv(uart_mb, (rt_ubase_t *)&data, RT_NO_WAIT) == RT_EOK) {
                // 安全区域：可以做任何事
                rt_kprintf("Received: 0x%02X\n", *data);
                rt_free(data);
            }
        }
    }
}

// ===== 上半部：ISR =====
void USART1_IRQHandler(void) {
    if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET) {
        uint8_t ch = USART_ReceiveData(USART1);
        
        uint8_t *buf = rt_malloc(sizeof(uint8_t));
        if (buf) {
            *buf = ch;
            if (rt_mb_send(uart_mb, (rt_ubase_t)buf) != RT_EOK) {
                rt_free(buf);  // 邮箱满了，丢弃数据
            }
        }
        
        rt_event_send(uart_event, UART_RX_EVT);
    }
}

// ===== 初始化 =====
int uart_app_init(void) {
    uart_event = rt_event_create("uart_evt", RT_IPC_FLAG_PRIO);
    uart_mb = rt_mb_create("uart_mb", 32, RT_IPC_FLAG_PRIO);
    
    uart_rx_thread = rt_thread_create("uart_rx",
                                       uart_rx_thread_entry,
                                       RT_NULL,
                                       1024,
                                       5,
                                       20);
    rt_thread_startup(uart_rx_thread);
    return 0;
}
```

---

## 五、延迟调度 —— "先记账，后结账"

### 5.1 完整时间线

```
时刻1：低优先级 logger 线程在执行
时刻2：按键按下 → EXTI 中断来了
时刻3：NVIC 让 ISR 执行（logger 线程被抢占）
时刻4：ISR 里调用 rt_event_send()
        → 把 UI 线程从"阻塞态"移到"就绪态"
        → 检查：UI 线程优先级 > logger 线程优先级
        → "记账"：需要调度（设 flag）
        → ISR 继续执行，不做任何耗时操作
时刻5：ISR 执行完，退出
        → rt_interrupt_leave() → nesting == 0
        → 检查 flag → 需要调度
        → 设置 PendSV 挂起
时刻6：PendSV 执行 → 切换到 UI 线程
时刻7：UI 线程从 rt_event_recv() 返回 → 执行业务逻辑
```

### 5.2 为什么不能在 ISR 里直接调度

```
ISR 执行到一半时直接调度：
  → MSP 上有 ISR 的临时数据未释放
  → 嵌套计数还是 > 0
  → PSP 切换后 ISR 的栈帧悬空
  → 系统状态断裂 → 崩溃

所以必须"延迟"：
  → ISR 里只做"记账"（改任务状态、设调度 flag）
  → ISR 正常退出（清理 MSP 上的临时数据）
  → 退出后检查 flag → 触发 PendSV → 安全切换
```

---

## 六、中断退出时的调度决策

```
ISR 执行完毕
  │
  ├── nesting--
  │
  ├── nesting == 0 ?
  │     │
  │     ├── 否 → 不调度，回到被中断的任务继续执行
  │     │
  │     └── 是 → 检查是否有更高优先级任务被唤醒
  │               │
  │               ├── 否 → 不调度，回到被中断的任务
  │               │
  │               └── 是 → 设置 PendSV 挂起
  │                           → 中断退出后执行 PendSV
  │                           → 切换到更高优先级任务
```

### 多个中断排队时的执行顺序

```
场景：任务C（优先级8）在执行
  同时发生：UART ISR（优先级5）、Timer ISR（优先级3）

NVIC 决策（只管中断优先级）：
  → Timer ISR(3) 先执行
  → UART ISR(5) 再执行
  → UART ISR 里唤醒任务A（优先级3）

RTOS 决策（管任务优先级）：
  → 任务A(3) > 任务C(8)
  → PendSV 切换到任务A

最终顺序：Timer ISR → UART ISR → 任务A
```

> [!tip] NVIC 中断调度和 RTOS 任务调度是两个独立的调度器，各管各的。NVIC 管"哪个中断先执行"，rt_schedule 管"哪个任务先跑"。

---

## 七、RT-Thread vs FreeRTOS 中断管理对比

| 方面 | RT-Thread | FreeRTOS |
|------|-----------|----------|
| 嵌套计数 | `rt_interrupt_nest` | `ulInterruptNesting` |
| 进入中断 | `rt_interrupt_enter()` | 编译器自动插入或手写 |
| 退出中断 | `rt_interrupt_leave()` → 内部调 `rt_schedule()` | `vPortExitInterrupt()` → `portYIELD_FROM_ISR()` |
| ISR 版 API | 统一 API，内部自动判断上下文 | 显式区分，FromISR 后缀 |
| 调度触发 | `rt_schedule()` 内部检查 nesting | `portYIELD_FROM_ISR()` 显式传入 woken flag |
| PendSV | `PendSV_Handler` 汇编 | `xPortPendSVHandler` 汇编 |
| 设计哲学 | 隐藏细节，统一接口 | 显式区分，API 数量翻倍但不会误用 |

### RT-Thread 统一 API 的内部实现

```c
rt_err_t rt_sem_release(rt_sem_t sem) {
    register int temp;
    temp = rt_hw_interrupt_disable();
    
    // 内部自动判断当前是在中断还是任务中
    if (rt_interrupt_get_nest() != 0) {
        // 中断中 → 不阻塞，直接操作
    } else {
        // 任务中 → 可以调度
    }
    
    rt_hw_interrupt_enable(temp);
    return RT_EOK;
}
```

---

## 八、上半部/下半部在 RTOS 中的实现对比

```
                    裸机              FreeRTOS            RT-Thread
───────────────────────────────────────────────────────────────────────
上半部（ISR）     设 flag           xTaskNotifyFromISR   rt_event_send
                                    xQueueSendFromISR    rt_mb_send
                                    xSemaphoreGiveFromISR rt_sem_release

下半部            main 循环         专门的处理任务        专门的处理线程
（Task/Thread）   轮询 flag         ulTaskNotifyTake     rt_event_recv
                                    xQueueReceive       rt_mb_recv

连接机制          flag 变量          任务通知 / 队列      事件集 / 邮箱 / 信号量
```

> [!tip] 这是通用设计模式。Linux 的 Top Half / Bottom Half、FreeRTOS 的 FromISR + Task、RT-Thread 的 ISR + 线程，都是同一思路。

---

## 九、数据风暴 —— 当中断来得太快

### 9.1 会发生什么

```
UART 每 1ms 来一帧数据，处理线程每帧需要 5ms 处理：

时刻1：ISR 收数据 → 塞队列 → 队列长度 1
时刻2：ISR 收数据 → 塞队列 → 队列长度 2
...
时刻33：ISR 收数据 → 队列满了！→ rt_mb_send 返回 err → 数据丢失！

后果：
  ① 队列/邮箱满后，后续数据直接丢弃
  ② 处理线程永远追不上 → 数据持续丢失
  ③ 处理线程一直就绪 → 抢占 CPU → 低优先级任务被饿死
```

### 9.2 解决方案

| 方案 | 做法 | 适用场景 |
|------|------|---------|
| 加大队列深度 | 增加缓冲空间 | 数据量偶尔突发 |
| 提高处理线程优先级 | 让它更快被调度执行 | 处理线程被其他任务抢占 |
| DMA + 环形缓冲区 | 减少中断频率 | 高速串口/SPI/ADC |
| 定时器轮询模式 | 不用中断，定时主动读取 | 极高频场景，避免中断风暴 |
| 降采样 | 不是每个数据都处理 | 传感器数据过密 |

> [!warning] 中断风暴：如果中断触发频率超过 CPU 处理能力，系统会陷入"进中断 → 退中断"的死循环，所有任务都被饿死。此时必须从硬件层面降低中断频率（DMA、硬件滤波）。

---

## 十、关键概念速查

| 概念 | 说明 |
|------|------|
| 中断嵌套计数 | 跟踪当前中断嵌套深度，nesting == 0 才允许调度 |
| FromISR | ISR 专用的 RTOS API 版本，绝不阻塞，只做记账 |
| 延迟调度 | ISR 中只设 flag，退出后才触发 PendSV 切换任务 |
| 上半部/下半部 | ISR 只做最少工作（上半部），重活交给任务（下半部） |
| IPC | 进程间通信，ISR 通过它传数据 + 唤醒任务 + 标记调度 |
| 中断风暴 | 中断频率过高导致系统无法正常调度 |

---

## 十一、面试高频问题

> [!example]- Q1：RTOS 中断和裸机中断的本质区别？
> 裸机靠轮询（main 循环一直在看 flag），RTOS 靠通知（ISR 通过 IPC 主动唤醒任务）。裸机只有一个执行流，ISR 设 flag main 下次循环就能看到。RTOS 多个任务可能在不同阻塞状态，普通 flag 它们看不到，必须用 IPC 机制同时完成"传数据 + 唤醒任务 + 标记调度"。

> [!example]- Q2：为什么 ISR 中不能用普通 RTOS API？
> 普通 API 可能阻塞（如队列满时等待）。ISR 没有任务上下文（没有 TCB），阻塞会导致系统死锁或崩溃。FromISR 版本绝不阻塞，失败直接返回错误，通过 woken flag 延迟调度。

> [!example]- Q3：延迟调度的本质是什么？
> "先记账，后结账"。ISR 中只做记账（改任务状态、设调度 flag），不做实际切换。等 ISR 正常退出、MSP 清理干净、嵌套计数归零后，再触发 PendSV 安全切换。本质是保证上下文切换在"干净的上下文"中发生。

> [!example]- Q4：多个中断排队且唤醒了高优先级任务，执行顺序？
> NVIC 按硬件优先级决定中断执行顺序（数字小先执行），与任务优先级无关。所有中断退出后，RTOS 按任务优先级决定切换到哪个任务。两个调度器各管各的。

> [!example]- Q5：中断数据来得太快处理不过来怎么办？
> 队列满后数据丢失，处理线程饿死低优先级任务。解决方案：加大队列、提高处理线程优先级、DMA + 环形缓冲区减少中断频率、极高频场景改用定时器轮询。

> [!example]- Q6：RT-Thread 和 FreeRTOS 的中断 API 设计有什么区别？
> FreeRTOS 显式区分 FromISR 版本，API 数量翻倍但不会误用。RT-Thread 统一 API 内部自动判断上下文（通过 rt_interrupt_get_nest()），使用更简单但内部逻辑更复杂。

---

## 十二、踩坑记录

> [!bug] 实战经验填充区
> （项目开发中遇到的 RTOS 中断相关问题记录于此）
