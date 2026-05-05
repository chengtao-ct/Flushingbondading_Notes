---
aliases:
  - SMP
  - 对称多处理
  - Symmetric Multiprocessing
tags:
  - 嵌入式
  - 硬件与芯片
  - SMP
  - 多核
date: 2026-04-28
status: ✅完成
related:
  - "[[_芯片架构总览]]"
  - "[[AMP架构]]"
  - "[[MCU架构]]"
  - "[[多核并发与Cache一致性]]"
---

> [!abstract] 核心定位
> SMP（对称多处理）让多个核心共享同一内存空间、运行同一操作系统，由 OS 统一调度。本文件聚焦 SMP 的核心技术：MESI 缓存一致性协议、同步原语、大小核架构、伪共享问题。

---

## 一、SMP的物理架构：多核如何"协同作战"？

### 1.1 核心矛盾：单核的性能瓶颈

单核处理器提升性能的三条路：
- **提高主频** → 功耗随频率立方增长，散热无解
- **超标量/流水线** → 复杂度爆炸，边际效益递减
- **多核并行** → **SMP的答案**

```mermaid
flowchart LR
    subgraph 单核困境
        A["主频↑"] --> B["功耗↑³"]
        B --> C["散热瓶颈"]
    end
    
    subgraph SMP方案
        D["核心数↑"] --> E["并行处理"]
        E --> F["主频不变<br/>性能线性提升"]
    end
```

---

### 1.2 SMP的物理布局：共享一切，平等竞争

```mermaid
flowchart TB
    subgraph SMP_System["SMP系统架构"]
        direction TB
        
        subgraph Cores["多核处理器集群"]
            direction LR
            Core0["CPU Core 0<br/>L1 I-Cache<br/>L1 D-Cache"]
            Core1["CPU Core 1<br/>L1 I-Cache<br/>L1 D-Cache"]
            Core2["CPU Core 2<br/>L1 I-Cache<br/>L1 D-Cache"]
            Core3["CPU Core 3<br/>L1 I-Cache<br/>L1 D-Cache"]
        end
        
        subgraph CacheCoherency["缓存一致性层"]
            L2["L2 Cache<br/>（共享或私有）"]
            Snoop["Snoop Controller<br/>监听控制器"]
            Dir["Directory<br/>目录协议"]
        end
        
        subgraph Interconnect["互连架构"]
            Bus["共享总线<br/>AXI/AHB"]
            NoC["片上网络<br/>Network-on-Chip"]
        end
        
        subgraph Shared["共享资源"]
            Mem["主存储器<br/>DDR"]
            Periph["外设<br/>GPIO/UART/SPI"]
            IntCtrl["中断控制器<br/>GIC"]
        end
    end
    
    Core0 <--> L2
    Core1 <--> L2
    Core2 <--> L2
    Core3 <--> L2
    
    L2 <--> Snoop
    Snoop <--> Dir
    
    L2 <--> Bus
    Bus <--> NoC
    
    Bus <--> Mem
    Bus <--> Periph
    Bus <--> IntCtrl
```

**SMP的核心定义：**
- 所有核心**地位平等**，无主从之分
- 所有核心**共享同一内存空间**，统一编址
- 所有核心**共享同一操作系统**，统一调度
- 所有核心**共享同一外设**，统一访问

---

> [!tip] SMP 与 AMP 的完整对比见 [[_芯片架构总览]]

---

### 1.4 缓存一致性：SMP的"心脏"

**问题场景：**

```c
// Core 0 执行
int shared_data = 100;
cache_flush();  // 数据在L1中

// Core 1 执行（稍后）
int local = shared_data;  // 从哪里读？L1？L2？内存？
```

```mermaid
sequenceDiagram
    participant C0 as Core 0
    participant L0 as L1 Cache (Core 0)
    participant L1 as L1 Cache (Core 1)
    participant C1 as Core 1
    participant Mem as 主内存
    participant Snoop as Snoop控制器
    
    Note over C0,Mem: Core 0 写入数据
    C0->>L0: 写入 shared_data = 100
    L0->>Snoop: 广播写操作
    Snoop->>L1: 使失效 Core 1 的副本
    Snoop->>Mem: 回写主存（可选）
    
    Note over C0,Mem: Core 1 读取数据
    C1->>L1: 读取 shared_data
    L1->>L1: Cache Miss!
    L1->>Snoop: 请求最新数据
    Snoop->>L0: 获取脏数据
    L0->>L1: 传递数据
    L1->>C1: 返回 100
```

**MESI协议状态机：**

```mermaid
stateDiagram-v2
    [*] --> Invalid
    
    Invalid --> Shared: 读操作<br/>（其他核有副本）
    Invalid --> Exclusive: 读操作<br/>（唯一副本）
    Invalid --> Modified: 写操作<br/>（独占写入）
    
    Shared --> Exclusive: 其他核<br/>放弃副本
    Shared --> Modified: 写操作
    Shared --> Invalid: 其他核写入
    
    Exclusive --> Modified: 写操作
    Exclusive --> Shared: 其他核读取
    Exclusive --> Invalid: 其他核写入
    
    Modified --> Shared: 其他核读取<br/>（回写并共享）
    Modified --> Invalid: 其他核写入<br/>（回写并失效）
```

| 状态 | 含义 | 可读 | 可写 |
|------|------|------|------|
| **M**odified | 已修改，仅本核持有 | ✓ | ✓ |
| **E**xclusive | 独占，未修改 | ✓ | ✓（无总线事务） |
| **S**hared | 共享，未修改 | ✓ | ✗（需总线仲裁） |
| **I**nvalid | 无效 | ✗ | ✗ |

---

## 二、SMP的设计哲学：透明并行，操作系统接管一切

### 2.1 负载均衡：操作系统的核心职责

```mermaid
flowchart LR
    subgraph 任务队列
        T1["Task 1"]
        T2["Task 2"]
        T3["Task 3"]
        T4["Task 4"]
        T5["Task 5"]
        T6["Task 6"]
    end
    
    subgraph 调度器
        Sched["OS Scheduler<br/>CFS/POSIX"]
    end
    
    subgraph 核心
        C0["Core 0<br/>运行 T1, T4"]
        C1["Core 1<br/>运行 T2, T5"]
        C2["Core 2<br/>运行 T3, T6"]
        C3["Core 3<br/>空闲"]
    end
    
    T1 --> Sched
    T2 --> Sched
    T3 --> Sched
    T4 --> Sched
    T5 --> Sched
    T6 --> Sched
    
    Sched --> C0
    Sched --> C1
    Sched --> C2
    Sched --> C3
```

**调度策略：**

| 策略 | 原理 | 适用场景 |
|------|------|----------|
| **CFS**（完全公平调度） | 红黑树按虚拟运行时间排序 | 通用Linux |
| **SCHED_FIFO** | 实时优先级队列，不抢占 | 实时任务 |
| **SCHED_RR** | 时间片轮转的实时调度 | 实时任务 |
| **Affinity** | 绑定任务到特定核心 | 缓存亲和性优化 |

---

### 2.2 同步原语：多核协作的"交通规则"

```c
// 自旋锁：忙等待，适用于短临界区
spinlock_t lock;
spin_lock(&lock);
// 临界区代码
spin_unlock(&lock);

// 互斥锁：可睡眠，适用于长临界区
struct mutex mtx;
mutex_lock(&mtx);
// 临界区代码
mutex_unlock(&mtx);

// 原子操作：无锁编程基础
atomic_t counter;
atomic_inc(&counter);  // 原子递增

// 内存屏障：防止指令重排
smp_wmb();  // 写内存屏障
smp_rmb();  // 读内存屏障
smp_mb();   // 全内存屏障
```

**自旋锁 vs 互斥锁：**

```mermaid
flowchart TD
    A[需要保护临界区] --> B{临界区时长？}
    
    B -->|极短<br/>< 100周期| C["自旋锁<br/>忙等待，不睡眠"]
    B -->|较长| D["互斥锁<br/>可睡眠，让出CPU"]
    
    C --> E{上下文？}
    E -->|中断上下文| F["spin_lock<br/>必须用自旋锁"]
    E -->|进程上下文| G["spin_lock<br/>可用，但注意持有时间"]
    
    D --> H["mutex_lock<br/>进程上下文专用"]
```

---

### 2.3 中断亲和性：中断分发到特定核心

```c
// Linux 设置中断亲和性
// 将中断IRQ 42 绑定到 Core 0 和 Core 1
echo "0-1" > /proc/irq/42/smp_affinity

// 代码中设置
struct cpumask mask;
cpumask_clear(&mask);
cpumask_set_cpu(0, &mask);
cpumask_set_cpu(1, &mask);
irq_set_affinity(42, &mask);
```

**设计考量：**
- **网络中断**：绑定到特定核心，提高缓存命中率
- **磁盘I/O**：分散到多核，避免单核瓶颈
- **实时任务**：独占核心，避免中断干扰

---

## 三、芯片选型：主流SMP处理器对比

### 3.1 主流SMP芯片对比

| 芯片 | 架构 | 核心数 | 主频 | 典型应用 | 特点 |
|------|------|--------|------|----------|------|
| **NXP i.MX8QM** | Cortex-A72+A53 | 2+4 | 1.6GHz | 汽车IVI、工业 | 大小核架构 |
| **RK3399** | Cortex-A72+A53 | 2+4 | 1.8GHz | 平板、边缘计算 | 国产化、高性价比 |
| **TI AM6548** | Cortex-A53 | 4 | 1.1GHz | 工业控制 | 集成PRU实时单元 |
| **NXP S32G2** | Cortex-A53 | 4 | 1.0GHz | 汽车网关 | ASIL-D功能安全 |
| **Intel Atom x6000** | x86 Tremont | 4 | 3.0GHz | 工业PC | x86生态兼容 |
| **SiFive U74-MC** | RISC-V U74 | 4 | 1.4GHz | AIoT | 开源架构 |

---

### 3.2 大小核架构：性能与功耗的平衡

```mermaid
flowchart TB
    subgraph BigLittle["大小核架构"]
        direction TB
        
        subgraph BigCluster["大核集群"]
            Big0["Cortex-A72<br/>高性能<br/>高功耗"]
            Big1["Cortex-A72"]
        end
        
        subgraph LittleCluster["小核集群"]
            Little0["Cortex-A53<br/>低功耗<br/>够用性能"]
            Little1["Cortex-A53"]
            Little2["Cortex-A53"]
            Little3["Cortex-A53"]
        end
        
        subgraph Scheduler["HMP调度器"]
            HMP["Heterogeneous<br/>Multi-Processing"]
        end
    end
    
    Big0 --> HMP
    Big1 --> HMP
    Little0 --> HMP
    Little1 --> HMP
    Little2 --> HMP
    Little3 --> HMP
    
    HMP --> Task["任务分配决策"]
    
    Task -->|"计算密集"| Big0
    Task -->|"后台任务"| Little0
```

**调度策略：**

| 任务类型 | 分配策略 | 原因 |
|----------|----------|------|
| UI渲染 | 大核 | 响应速度优先 |
| 视频编解码 | 大核 | 计算密集 |
| 后台同步 | 小核 | 功耗优先 |
| 音频播放 | 小核 | 计算量小 |
| 游戏 | 大核+小核 | 混合负载 |

---

## 四、嵌入式工程应用：SMP的实际战场

### 4.1 典型应用场景

```mermaid
mindmap
  root((SMP应用))
    汽车电子
      智能座舱
      自动驾驶域控
      网关
    工业控制
      运动控制
      视觉检测
      边缘计算
    消费电子
      智能手机
      平板电脑
      智能音箱
    通信设备
      基站
      路由器
      交换机
```

### 4.2 实战案例：汽车智能座舱

i.MX8QM（2×A72 + 4×A53）的大小核任务分配——通过 CPU affinity 将任务绑定到合适的核心：

```c
// 大核 A72: UI渲染、语音识别、导航（计算密集）
CPU_SET(0, &cpuset); CPU_SET(1, &cpuset);
pthread_setaffinity_np(ui_thread, sizeof(cpuset), &cpuset);

// 小核 A53: 日志、OTA、诊断（后台低功耗）
CPU_SET(2, &cpuset); CPU_SET(3, &cpuset);
pthread_setaffinity_np(log_thread, sizeof(cpuset), &cpuset);
```

---

### 4.3 实战案例：工业视觉检测

多核并行图像处理——按行分割图像，每核处理一个区域：

```c
void parallel_image_process(uint8_t *img, int h, int w) {
    int cores = sysconf(_SC_NPROCESSORS_ONLN);
    for (int i = 0; i < cores; i++) {
        tasks[i] = (ImageTask){img, i*h/cores, (i+1)*h/cores, w};
        pthread_create(&threads[i], NULL, process_region, &tasks[i]);
    }
    for (int i = 0; i < cores; i++) pthread_join(threads[i], NULL);
}
```

关键设计：按行分割避免跨核数据依赖、pthread 并行处理。

---

## 五、大师的工程建议

### 5.1 SMP开发的核心陷阱

| 陷阱 | 表现 | 根因 | 解决方案 |
|------|------|------|----------|
| **伪共享** | 多核性能反而下降 | 不同核频繁修改同一缓存行 | 对齐到缓存行大小（64字节） |
| **锁竞争** | 核心空转，CPU利用率低 | 锁粒度过粗 | 细粒度锁、RCU、无锁数据结构 |
| **缓存一致性风暴** | 总线带宽耗尽 | 频繁跨核数据访问 | 数据局部性优化、NUMA感知 |
| **优先级反转** | 高优先级任务被阻塞 | 锁被低优先级任务持有 | 优先级继承协议 |
| **负载不均衡** | 部分核心过载 | 调度策略不当 | 手动设置亲和性、cgroups |

---

### 5.2 伪共享问题详解

```c
// 问题代码：伪共享
struct Counter {
    int value;  // 4字节
} __attribute__((packed));

struct Counter counters[4];  // 4个计数器，可能共享缓存行

// 多核各自递增
void increment_counter(int id) {
    counters[id].value++;  // 导致整个缓存行失效
}

// 修复：缓存行对齐
struct Counter {
    int value;
    char padding[60];  // 填充到64字节
} __attribute__((aligned(64)));

// 或使用宏
#define CACHE_LINE_SIZE 64
struct Counter {
    int value;
} __attribute__((aligned(CACHE_LINE_SIZE)));
```

```mermaid
flowchart TB
    subgraph 问题["伪共享问题"]
        direction LR
        C0["Core 0"] -->|"修改 counter[0]"| CL1["缓存行<br/>包含 counter[0-3]"]
        C1["Core 1"] -->|"修改 counter[1]"| CL1
        C2["Core 2"] -->|"修改 counter[2]"| CL1
        C3["Core 3"] -->|"修改 counter[3]"| CL1
        
        Note1["每次修改导致整个缓存行<br/>在核心间来回传递"]
    end
    
    subgraph 修复["缓存行对齐后"]
        direction LR
        C0_["Core 0"] --> CL0_["缓存行0<br/>counter[0]"]
        C1_["Core 1"] --> CL1_["缓存行1<br/>counter[1]"]
        C2_["Core 2"] --> CL2_["缓存行2<br/>counter[2]"]
        C3_["Core 3"] --> CL3_["缓存行3<br/>counter[3]"]
        
        Note2["每个计数器独占缓存行<br/>无跨核失效"]
    end
```

---

## 总结

| 维度 | 核心特点 |
|------|----------|
| 物理架构 | 共享内存、统一编址、MESI 缓存一致性协议 |
| 设计哲学 | OS 统一调度、负载均衡、透明并行 |
| 核心挑战 | 同步与锁、伪共享、缓存一致性风暴 |

> [!quote] 本质
> SMP 不是"多个 CPU 拼在一起"，而是一套由硬件缓存一致性 + OS 调度器共同支撑的并行计算生态系统。

## 知识拓扑

- 上层：[[_芯片架构总览]] — SMP 在处理器全景中的定位
- 对比：[[AMP架构]] — 非对称多核的设计哲学差异
- 深入：[[多核并发与Cache一致性]] — MESI 协议与内存屏障详解
- 前置：[[MCU架构]] — 单核架构是理解多核的基础