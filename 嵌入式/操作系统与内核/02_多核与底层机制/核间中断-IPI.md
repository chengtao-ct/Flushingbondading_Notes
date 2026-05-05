---
aliases: [IPI, Inter-Processor Interrupt, 核间通信]
tags:
  - 嵌入式
  - 多核
  - 中断
related:
  - "[[多核并发与Cache一致性]]"
date: 2026-01-27
status: 🌿草稿
chip: Generic
---
核间中断（IPI，Inter-Processor Interrupt）是多核处理器提供的一种纯硬件通信机制；在 RT-Thread 中，它的核心作用是打破核心之间的物理屏障，允许一个 CPU 核心强行打断另一个 CPU 核心的执行流，强制它立刻进行上下文切换（任务调度）。

##### 为什么 Core 1 不能自己去发现新任务？

如果不用 IPI，Core 1 只有等到它的"滴答定时器"也就是时间片耗尽时，才会去检查就绪队列。假设 Tick 是 1 毫秒，那么这个最高优先级的急停任务最多可能要等 1 毫秒才会被 Core 1 执行。在实时操作系统中，1 毫秒的延迟足以让电机撞毁机器。

**IPI 的存在，就是为了保证"最高优先级任务被唤醒后，能在微秒级时间内立刻抢占任何核心的 CPU"，这捍卫了 RTOS 绝对的"硬实时性"。**

```mermaid
sequenceDiagram
    participant Core0 as 核心0
    participant ReadyQ as 就绪队列
    participant Core1 as 核心1
    
    Core0->>ReadyQ: 创建高优先级任务
    Note over ReadyQ: 任务进入就绪队列
    Core0->>Core1: 发送核间中断 IPI
    Core1->>Core1: 立即响应中断
    Core1->>ReadyQ: 检查就绪队列
    Core1->>Core1: 触发任务调度
    Note over Core1: 切换到高优先级任务<br/>延迟 < 10us
    
    Note over Core1: 对比：无 IPI 时
    Core1->>Core1: 等待 Tick 到期
    Note over Core1: 延迟可达 1ms
```





