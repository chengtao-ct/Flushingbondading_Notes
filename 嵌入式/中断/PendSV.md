---
tags:
  - 嵌入式
  - 中断
  - RTOS
  - ARM-Cortex-M
aliases:
  - PendSV
  - Pendable Service Call
  - 可悬挂系统调用
  - 任务切换
  - Context Switch
related:
  - "[[栈与上下文]]"
  - "[[中断的基础理解]]"
  - "[[向量表的基础理解]]"
  - "[[../操作系统与内核/04_FreeRTOS/内存管理/中断推迟处理]]"
  - "[[../源码阅读/RT-thread源码阅读-v2/05-中断-Tick-定时器]]"
chip: STM32
---

# PendSV 与任务切换

> [!abstract]
> PendSV 是 Cortex-M 专为 RTOS 任务切换设计的系统异常。优先级设为最低（255），保证任务切换永远不打断任何硬件中断。所有任务切换最终都汇聚到 PendSV 这一个入口执行。

> [!info] 面试开场句
> "PendSV 是 Cortex-M 的可悬挂系统调用，RTOS 把它的优先级设为最低，保证所有硬件中断先处理完再切换任务。切换过程：保存旧任务 R4-R11 → 切换 SP → 恢复新任务 R4-R11 → BX LR 触发硬件出栈 → CPU 飞到新任务。"

---

## 一、PendSV 是什么

PendSV 全称 **Pendable Service Call**（可悬挂的服务调用），是 Cortex-M 向量表中的系统异常之一：

```
向量表中的系统异常（部分）：
  Reset          → 上电复位
  NMI            → 不可屏蔽中断
  HardFault      → 硬件错误
  SVCall         → 系统服务调用（软件触发，立即执行）
  PendSV         → 可悬挂的系统调用（挂起，等 CPU 空了再执行）← 这个
  SysTick        → 系统滴答定时器
```

"可悬挂"的含义：
- 普通中断来了就立刻执行
- **PendSV 可以被"挂起"（Pending），等所有更高优先级的中断处理完后才执行**

触发方式：

```c
// 设置 PendSV 挂起标志
SCB->ICSR = SCB_ICSR_PENDSVSET_Msk;

// 写完这行，PendSV 不会立刻执行
// NVIC 会等所有更高优先级的中断处理完后才让 PendSV 执行
```

---

## 二、为什么 RTOS 用 PendSV 做任务切换

这是面试必考题。PendSV 的核心优势：**优先级设为最低，保证任务切换不打断任何硬件中断。**

### 2.1 为什么不用其他机制

```
方案1：在 SysTick 里直接切任务
  → SysTick 优先级可以设低，但无法保证"所有硬件中断都处理完"
  → SysTick 执行时可能有其他中断正在排队
  → 在 SysTick 里切任务会干扰其他中断的正常响应

方案2：在高优先级 ISR 里直接切任务
  → 高优先级中断可能打断低优先级中断
  → 在中断上下文中做任务切换 = Q2/栈与上下文 中讲的"ISR 中途不能切任务"

方案3：用 SVCall 做任务切换
  → SVCall 是立即执行的，优先级可以设很高
  → 但 RTOS 需要的是"最晚执行"的切换，不是"最高优先级"
  → SVCall 适合紧急的系统调用，不适合延迟的任务切换
```

### 2.2 PendSV 的独特优势

```
时间线：

  任务A 执行中
    │
    ├── SysTick 中断来了
    │   → NVIC 让 SysTick 执行（优先级比 PendSV 高）
    │   → 更新 tick、检查时间片
    │   → 发现该切换任务了
    │   → 设置 PendSV 挂起标志
    │   → SysTick 正常退出
    │
    ├── UART 中断也在排队
    │   → NVIC 让 UART 先执行（优先级比 PendSV 高）
    │   → UART 处理完退出
    │
    ├── 所有更高优先级中断都处理完了
    │
    └── PendSV 终于执行（优先级最低，最后才轮到）
        → 做任务切换 → 切换到任务B

关键：任务切换不会打断任何硬件中断的执行！
```

---

## 三、两层调度：NVIC vs RTOS

这是理解 PendSV 的关键 —— 不要把 NVIC 的中断调度和 RTOS 的任务调度混在一起：

```
第一层：NVIC（硬件中断调度）
  管的是"哪个中断先执行"
  基于硬件优先级：数字越小优先级越高
  UART(5) > SysTick(15) > PendSV(255)

第二层：rt_schedule（软件任务调度）
  管的是"哪个任务先跑"
  基于任务优先级：数字越小优先级越高
  任务B(优先级3) > 任务A(优先级5)

PendSV 是两层之间的桥梁：
  它是一个中断（归 NVIC 管）
  但它里面执行的是任务切换（归 RTOS 管）
```

### 完整闭环

```
调度决策 = rt_schedule()（RTOS 软件，判断该切谁）
    │
    ▼
触发切换 = 设置 PendSV 挂起标志（软件动作）
    │
    ▼
等待时机 = NVIC 保证所有硬件中断先执行完（硬件优先级机制）
    │
    ▼
执行切换 = PendSV ISR 里换栈换寄存器（软硬件配合）
```

> [!important] 任务切换 = PendSV 这一条路。CPU 硬件只做压栈/出栈，不知道什么是"任务"。RTOS 利用了 CPU 的中断压栈/出栈机制来实现任务切换。PendSV 是所有任务切换的"唯一执行者"，上层 API（yield / delay / ISR 通知）都只是"触发者"。

---

## 四、任务切换的完整过程

### 4.1 切换前的状态

```
任务A 正在执行（用 PSP_A）

TCB_A: { sp = ???, priority = 5, ... }   ← sp 记录任务A 的栈指针
TCB_B: { sp = 伪造的栈帧地址, priority = 3, ... }  ← 任务B 还没运行过
```

### 4.2 触发 PendSV

```
SysTick 中断 → rt_tick_increase() → rt_schedule() 发现任务B 优先级更高
→ 设置 PendSV 挂起：SCB->ICSR = SCB_ICSR_PENDSVSET_Msk
→ SysTick 正常退出
→ NVIC 等所有更高优先级中断处理完
→ PendSV 执行
```

### 4.3 PendSV ISR 执行（汇编）

```asm
PendSV_Handler:
    ; ① 禁止中断（防止 R4-R11 保存/恢复被打断）
    CPSID   I

    ; ② 获取当前任务的 PSP（硬件已自动压了 8 个寄存器到 PSP 指向的栈）
    MRS     R0, PSP

    ; ③ 手动保存旧任务（任务A）的 R4-R11 到它的栈上
    ;    硬件已经保存了 R0-R3, R12, LR, PC, xPSR
    ;    现在手动保存 R4-R11（callee-saved 寄存器）
    STMDB   R0!, {R4-R11}         ; 压入 R4-R11，R0 更新为新栈顶

    ; ④ 把新的栈顶保存到任务A 的 TCB 中
    LDR     R1, =rt_current_thread
    LDR     R1, [R1]              ; R1 = 当前任务 TCB 指针
    STR     R0, [R1]              ; TCB_A.sp = 新的栈顶

    ; ⑤ 调用调度器，找到最高优先级任务（任务B）
    BL      rt_schedule           ; 更新 rt_current_thread 指向任务B

    ; ⑥ 从任务B 的 TCB 取出栈指针
    LDR     R1, =rt_current_thread
    LDR     R1, [R1]              ; R1 = 任务B 的 TCB 指针
    LDR     R0, [R1]              ; R0 = TCB_B.sp（伪造的栈帧地址）

    ; ⑦ 从任务B 的栈弹出 R4-R11
    LDMIA   R0!, {R4-R11}         ; 恢复 R4-R11，R0 更新

    ; ⑧ 把 R0 设为 PSP（指向伪造的 8 个硬件寄存器）
    MSR     PSP, R0

    ; ⑨ 开中断
    CPSIE   I

    ; ⑩ 返回
    ;    BX LR → CPU 识别 EXC_RETURN → 触发硬件出栈
    ;    从伪造栈帧弹出 R0-R3, R12, LR, PC, xPSR
    ;    PC 弹出 = 任务B 的入口函数 → 任务B 开始执行！
    BX      LR
```

### 4.4 栈帧变化图

```
任务A 的栈（被切走时）：           任务B 的栈（伪造的，首次启动）：
┌──────────┐                     ┌──────────┐
│ xPSR     │ ← 硬件自动压的      │ xPSR 0x01000000 │ ← 伪造的
│ PC       │   PC = 任务A被打断   │ PC = entry_B    │ ← 伪造的入口函数
│          │   时下一条指令地址    │                 │
│ LR       │                     │ LR = 0xFFFFFFFF │
│ R12      │                     │ R12             │
│ R3       │                     │ R3              │
│ R2       │                     │ R2              │
│ R1       │                     │ R1 = parameter  │
│ R0       │                     │ R0              │
├──────────┤ ← PendSV 手动压的   ├─────────────────┤ ← PendSV 手动弹的
│ R11      │                     │ R11             │
│ R10      │                     │ R10             │
│ R9       │                     │ R9              │
│ R8       │                     │ R8              │
│ R7       │                     │ R7              │
│ R6       │                     │ R6              │
│ R5       │                     │ R5              │
│ R4       │                     │ R4              │
└──────────┘                     └────────────────┘
  ↑ TCB_A.sp 指向这里              ↑ TCB_B.sp 指向这里
```

### 4.5 PC 值的含义

| 场景 | 栈帧中 PC 的值 | 含义 |
|------|--------------|------|
| 任务第一次被切走（之前一直在跑） | 任务被打断时**下一条指令**的地址 | 硬件自动压入，下次恢复从这里继续 |
| 任务首次启动（伪造栈帧） | 任务的**入口函数**地址 | 手动伪造，CPU "以为"从中断返回到入口 |
| 任务被切走后再次被切回来 | 上次被切走时保存的 PC | 从上次断点继续执行 |

---

## 五、PendSV vs SVCall

| | PendSV | SVCall |
|--|--------|--------|
| 触发方式 | 写 ICSR 寄存器（`SCB->ICSR = PENDSVSET`） | 执行 SVC 指令（`SVC #0`） |
| 执行时机 | 挂起，等所有更高优先级中断处理完 | 立即执行 |
| 典型优先级 | 最低（255） | 可设为任意优先级 |
| 用途 | 任务切换（延迟、不紧急） | 系统调用（立即、紧急） |
| 典型场景 | tick 到期后切换任务 | 任务主动请求内核服务 |
| RT-Thread | PendSV 做任务切换 | SVC 用于进入线程模式 |
| FreeRTOS | PendSV 做任务切换 | 极少使用 |

**一句话：SVCall 是"主动请求"（立即），PendSV 是"被动安排"（延迟）。**

---

## 六、FreeRTOS vs RT-Thread 的 PendSV 实现

两者原理完全一样，差异在调度算法，不在上下文切换：

```
FreeRTOS 的 PendSV_Handler（简化）：
  ① CPSID I
  ② 保存 R4-R11 到当前任务栈
  ③ 把当前 SP 存入当前任务 TCB
  ④ 调用 vTaskSwitchContext() → 找最高优先级就绪任务
  ⑤ 从新任务 TCB 取出 SP
  ⑥ 恢复 R4-R11
  ⑦ CPSIE I
  ⑧ BX LR

RT-Thread 的 PendSV_Handler（简化）：
  ① CPSID I
  ② 保存 R4-R11 到当前任务栈
  ③ 把当前 SP 存入当前任务 TCB
  ④ 调用 rt_schedule() → 找最高优先级就绪线程
  ⑤ 从新线程 TCB 取出 SP
  ⑥ 恢复 R4-R11
  ⑦ CPSIE I
  ⑧ BX LR
```

> [!tip] 几乎一模一样，因为都是 ARM Cortex-M 的标准做法。差异在调度算法（如何找最高优先级任务），不在上下文切换本身。

---

## 七、为什么 PendSV ISR 要关中断

```asm
CPSID   I    ; 第一步就关中断
```

**R4-R11 的保存/恢复必须是原子的**，不能被打断：

```
如果不关中断，PendSV 正在从新栈弹出 R4-R11：
  R4 = 新任务值 ✅ 已弹出
  R5 = 新任务值 ✅ 已弹出
  R6-R11 还没弹出

此刻 UART 中断打断：
  → UART ISR 可能触发新的任务调度
  → PendSV 被再次挂起
  → 双重上下文切换 → 栈帧彻底混乱 → 系统崩溃

危险的不是"共用 R4-R11"（编译器会自动保护 callee-saved 寄存器）
危险的是"被打断后再次触发任务切换"（导致双重上下文切换）
```

### NMI 能打断 PendSV 吗？

能。NMI 是不可屏蔽中断，`CPSID I` 关不掉。但 **NMI 安全**，因为：

```
NMI 打断 PendSV：
  → 硬件在 MSP 上压一层 8 个寄存器（中断嵌套）
  → NMI ISR 执行（编译器自动保存它用到的 R4-R11）
  → NMI 返回，自动出栈
  → PendSV 从断点继续
  → 一切正常

关键：RTOS 永远不会在 NMI 里做任务调度
  → PSP 没被动过
  → 不会触发第二次 PendSV
  → 所以安全
```

> [!warning] 关中断（CPSID I）防的是普通中断触发二次调度，不是防 R4-R11 被修改。R4-R11 的保护由编译器的 callee-saved 约定完成。

---

## 八、关键概念速查

| 概念 | 说明 |
|------|------|
| PendSV | 可悬挂系统调用，优先级最低，专为 RTOS 任务切换设计 |
| 挂起（Pending） | 设置标志但不立即执行，等更高优先级中断处理完 |
| 任务切换 | 保存旧任务上下文 + 切换 SP + 恢复新任务上下文 |
| CPSID I | 关中断，保证 R4-R11 保存/恢复的原子性 |
| SVCall | 立即执行的系统调用，与 PendSV（延迟）互补 |
| NVIC | 硬件中断调度器，管"哪个中断先执行" |
| rt_schedule | 软件任务调度器，管"哪个任务先跑" |

---

## 九、面试高频问题

> [!example]- Q1：为什么 RTOS 用 PendSV 做任务切换？
> PendSV 优先级设为最低（255），保证所有硬件中断先处理完再切换任务。这样任务切换不会打断任何硬件中断的执行。其他机制要么会干扰中断响应（在 ISR 里直接切），要么不够灵活（SysTick 里直接切）。

> [!example]- Q2：PendSV ISR 里为什么第一步要关中断？
> R4-R11 的保存和恢复必须是原子的。如果不关中断，一个普通中断（如 UART）可能打断 PendSV 并触发新的任务调度，导致双重上下文切换，栈帧彻底混乱。关中断防的是"二次调度"，不是防 R4-R11 被修改（编译器 callee-saved 约定已经保护了）。

> [!example]- Q3：NVIC 中断调度和 RTOS 任务调度有什么区别？
> NVIC 管的是"哪个中断先执行"，基于硬件优先级。RTOS 的 rt_schedule 管的是"哪个任务先跑"，基于任务优先级。PendSV 是两者的桥梁：它是 NVIC 管理的中断，但里面执行的是 rt_schedule 的调度决策。

> [!example]- Q4：NMI 打断 PendSV 会出问题吗？
> 不会。NMI 虽然不可屏蔽（CPSID I 关不掉），但 RTOS 永远不会在 NMI 里做任务调度，所以不会触发二次 PendSV。NMI ISR 用到的 R4-R11 由编译器自动保存恢复。NMI 正常返回后 PendSV 从断点继续，一切正常。

> [!example]- Q5：任务第一次被切走时，栈帧中 PC 的值是什么？
> 是任务被打断时**下一条指令**的地址，由硬件自动压入。不是入口函数。入口函数只在伪造栈帧（任务首次启动）时才作为 PC 的值。之后任务一直在跑，PC 就是"执行到了哪里"。

> [!example]- Q6：FreeRTOS 和 RT-Thread 的 PendSV 实现有什么区别？
> 上下文切换部分几乎一模一样：都是保存 R4-R11 → 切 SP → 恢复 R4-R11 → BX LR。差异在调度算法（如何找最高优先级任务），不在切换机制本身。

---

## 十、踩坑记录

> [!bug] 实战经验填充区
> （项目开发中遇到的 PendSV / 任务切换相关问题记录于此）
