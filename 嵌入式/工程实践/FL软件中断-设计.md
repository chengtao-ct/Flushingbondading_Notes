# FL软件中断编码器设计文档

## 1. 核心原理：软件中断编码的本质

### 1.1 编码器信号的物理来源

无论是硬件编码器还是软件中断编码器，**脉冲信号永远来自物理世界**：

```
电机转动 → 光栅/磁环旋转 → 光电/霍尔传感器 → A/B正交方波 → MCU引脚
```

软件中断编码器**不产生脉冲**，而是**响应脉冲**——用CPU执行ISR代码来替代硬件计数器。

### 1.2 硬件编码器 vs 软件中断编码器

|对比项|硬件编码器 (TIM QEI)|软件中断编码器 (EXTI)|
|---|---|---|
|计数执行者|TIM硬件电路（门电路+触发器）|CPU执行ISR代码|
|CPU参与度|零（后台自动运行）|必须（每次边沿触发中断）|
|计数介质|CNT寄存器|内存变量 `fl_soft_accum_4`|
|响应速度|纳秒级（纯硬件）|微秒级（中断响应+代码执行）|
|适用频率|MHz级别|<100kHz（受CPU负载限制）|
|资源代价|占用TIM外设|占用CPU时间 + EXTI通道|
|诊断能力|有限（仅CNT/DIR寄存器）|可扩展（非法跳变、抖动统计）|

### 1.3 软件编码的本质代价

软件中断编码器是用**CPU时间**换取**硬件资源**：

```
每次边沿触发ISR的开销 ≈ 20~50个时钟周期

假设：编码器1000线 × x4倍频 = 4000脉冲/转
      电机转速 3000 RPM = 50转/秒
      脉冲频率 = 200 kHz

CPU负载估算 = 200000 × 30周期 / 168MHz ≈ 3.6%
```

**工程边界**：软件中断编码器适用于**100kHz以下**的脉冲频率。超过此频率，丢步风险急剧上升。

---

## 2. 中断链路深度剖析

### 2.1 完整调用链

```
EXTI IRQ → HAL_GPIO_EXTI_IRQHandler → HAL_GPIO_EXTI_Callback → Encoder_OnFlaEdgeIsr
   │              │                          │                         │
   │              │                          │                         │
 硬件层        HAL库入口层                 解耦回调层                 业务逻辑层
```

### 2.2 各层职责与关键点

#### 第一层：EXTI IRQ（硬件中断请求）

- **触发条件**：PE5/PE6引脚检测到边沿（双边沿触发）
- **中断向量**：PE5~PE9 共用 `EXTI9_5_IRQn`（STM32F407）
- **硬件行为**：EXTI控制器自动将挂起位（PR）置1，向NVIC发送中断请求
- **关键认知**：A/B两相共用一个中断向量，ISR中必须**同时采样两路状态**

#### 第二层：HAL_GPIO_EXTI_IRQHandler（HAL库入口）

- **位置**：`stm32f4xx_it.c`
- **职责**：
    - 检查PR寄存器，判断是Line 5还是Line 6触发
    - 写1清零PR标志位
    - 调用对应的Callback函数
- **性能隐患**：HAL库内部有参数检查和保护逻辑，对于高频信号可能成为瓶颈

#### 第三层：HAL_GPIO_EXTI_Callback（弱定义回调）

- **机制**：利用链接器 `__weak` 特性实现解耦
- **参数**：`uint16_t GPIO_Pin`，区分触发源
- **设计原则**：仅做**快速分发**，不写复杂逻辑

#### 第四层：Encoder_OnFlaEdgeIsr（业务状态机）

- **核心任务**：采样AB状态 → 查表判断方向 → 累加计数
- **执行约束**：
    - 禁止浮点运算、除法、内存申请
    - 禁止调用阻塞函数（HAL_Delay等）
    - 禁止调用非重入函数
    - 目标执行时间 < 1μs

### 2.3 中断优先级配置

```c
// EXTI9_5_IRQn 必须有足够高的优先级
// 但不能抢占SysTick（RTOS依赖）
HAL_NVIC_SetPriority(EXTI9_5_IRQn, 1, 0);  // 抢占优先级1，子优先级0
HAL_NVIC_EnableIRQ(EXTI9_5_IRQn);
```

---

## 3. 状态机算法设计

### 3.1 正交编码原理

A/B两相相位差90°，一个完整周期包含4种状态：

```
        ┌───────┐       ┌───────┐
A:  ────┘       └───────┘       └────
          ┌───────┐       ┌───────┐
B:  ──────┘       └───────┘       └──

状态序列（顺时针）：
00 → 01 → 11 → 10 → 00 → ...
```

### 3.2 状态转移表

状态集合：

|prev\curr|00|01|11|10|
|---|---|---|---|---|
|**00**|0|+1|INV|-1|
|**01**|-1|0|+1|INV|
|**11**|INV|-1|0|+1|
|**10**|+1|INV|-1|0|

**转移含义**：

- `+1`：顺时针旋转1/4周期
- `-1`：逆时针旋转1/4周期
- `0`：重复采样（无变化）
- `INV`：非法跳变（跨两位，可能是干扰或采样丢失）

### 3.3 代码实现

```c
// 状态转移表（查表法，O(1)时间复杂度）
static const int8_t TRANSITION_TABLE[4][4] = {
    // 00   01   11   10  (curr)
    {  0,  +1,  -2,  -1}, // prev = 00
    { -1,   0,  +1,  -2}, // prev = 01
    { -2,  -1,   0,  +1}, // prev = 11
    { +1,  -2,  -1,   0}  // prev = 10
};

#define STEP_INVALID (-2)

void Encoder_OnFlaEdgeIsr(uint16_t GPIO_Pin)
{
    // 1. 原子采样AB状态（一次性读取，避免时序错位）
    uint8_t curr_ab = (GPIOE->IDR >> 5) & 0x03;
    
    // 2. 首次运行初始化
    if (!fl_prev_ab_valid) {
        fl_prev_ab = curr_ab;
        fl_prev_ab_valid = 1;
        return;
    }
    
    // 3. 查表获取步进值
    int8_t step = TRANSITION_TABLE[fl_prev_ab][curr_ab];
    
    // 4. 状态处理
    fl_isr_edge_total++;
    
    if (step > 0) {
        fl_soft_accum_4 += step;
        fl_valid_transition_count++;
    } else if (step < 0 && step != STEP_INVALID) {
        fl_soft_accum_4 += step;  // step = -1
        fl_valid_transition_count++;
    } else if (step == STEP_INVALID) {
        fl_invalid_transition_count++;
    }
    // step == 0: 重复采样，忽略
    
    // 5. 更新状态
    fl_prev_ab = curr_ab;
}
```

---

## 4. Tick10ms 融合策略

### 4.1 数据流

```
ISR层（异步）              Tick层（同步10ms）
    │                           │
    ▼                           ▼
fl_soft_accum_4  ────────►  原子读取并清零
    │                           │
    │                           ▼
    │                    fl_delta_4 × 极性
    │                           │
    │                           ▼
    │                    sample[FL].delta_count_4
    │                           │
    │                           ▼
    │                    pos_count_4 += delta
    │                           │
    │                           ▼
    │                    速度计算（Q16位移换算）
    │                           │
    │                           ▼
    │                    IIR滤波 → speed_mmps_filt
```

### 4.2 临界区保护

```c
void Encoder_Tick10ms(void)
{
    int32_t fl_delta_4;
    
    // 进入临界区，禁止中断
    __disable_irq();
    fl_delta_4 = fl_soft_accum_4;
    fl_soft_accum_4 = 0;
    __enable_irq();
    
    // 应用极性
    fl_delta_4 *= ENCODER_FL_POLARITY;
    
    // 写入采样缓冲
    sample[FL].delta_count_4 = fl_delta_4;
    
    // 位置累加
    sample[FL].pos_count_4 += fl_delta_4;
    
    // 速度计算（与硬件轮共用流程）
    // ... Q16位移换算 → speed_mmps_raw → IIR滤波 ...
}
```

---

## 5. 诊断可观测性设计

### 5.1 诊断快照结构

```c
typedef struct {
    uint32_t isr_edge_total;          // ISR总调用次数
    uint32_t valid_transition_count;  // 有效转移次数
    uint32_t invalid_transition_count; // 非法跳变次数
    uint32_t accum_overflow_count;    // 累加器溢出次数（可选）
} encoder_diag_snapshot_t;
```

### 5.2 诊断指标解读

|指标|正常范围|异常指示|
|---|---|---|
|`valid / edge`|≈ 100%|< 95% 表示信号质量差|
|`invalid_count`|接近0|持续增长表示线束干扰或接触不良|
|`invalid_count` 突增|-|电机启动/停止瞬间的电磁干扰|

### 5.3 API接口

```c
encoder_status_t Encoder_GetDiagSnapshot(encoder_wheel_t wheel, 
                                          encoder_diag_snapshot_t *out);
```

---

## 6. 去抖策略

### 6.1 V1：合法转移过滤（默认启用）

- **原理**：通过状态转移表，只接受相邻状态转移，拒绝跨位跳变
- **优点**：零时间开销，纯逻辑过滤
- **局限**：无法过滤单边沿抖动

### 6.2 V2：时间门限过滤（预留）

- **原理**：两次有效边沿间隔小于阈值则忽略
- **代价**：需要引入时间戳，增加ISR开销
- **适用场景**：极端干扰环境

### 6.3 V3：多次采样一致性（预留）

- **原理**：连续采样N次，结果一致才认为有效
- **代价**：显著增加ISR执行时间
- **适用场景**：低速高可靠性场景

---

## 7. 初始化与复位

### 7.1 初始化流程

```c
void Encoder_Init(void)
{
    // 1. GPIO初始化（EXTI双边沿触发）
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = GPIO_PIN_5 | GPIO_PIN_6;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;  // 编码器通常开路输出
    HAL_GPIO_Init(GPIOE, &GPIO_InitStruct);
    
    // 2. 中断优先级配置
    HAL_NVIC_SetPriority(EXTI9_5_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(EXTI9_5_IRQn);
    
    // 3. 软件状态初始化
    fl_prev_ab_valid = 0;
    fl_soft_accum_4 = 0;
    fl_isr_edge_total = 0;
    fl_valid_transition_count = 0;
    fl_invalid_transition_count = 0;
}
```

### 7.2 复位流程

```c
void Encoder_ResetWheel(encoder_wheel_t wheel)
{
    if (wheel == FL) {
        __disable_irq();
        fl_soft_accum_4 = 0;
        fl_prev_ab_valid = 0;
        // 诊断计数可选是否清零
        __enable_irq();
    }
}
```

---

## 8. 验收标准

|测试项|预期结果|
|---|---|
|正向旋转一圈|`fl_soft_accum_4` 累计 +4n（n为编码器线数）|
|反向旋转一圈|`fl_soft_accum_4` 累计 -4n|
|静止状态|`invalid_transition_count` 不增长|
|信号干扰|`invalid_transition_count` 增长，但 `fl_soft_accum_4` 不受影响|
|高速运行|CPU负载 < 10%，无丢步|
|低速换向|`speed_mmps_filt` 连续稳定，可用于控制环|

---

## 9. 工程避坑指南

### 9.1 中断响应延迟

- **现象**：高速时丢步
- **排查**：示波器测量引脚边沿到ISR入口的时间
- **解决**：提高中断优先级，或绕过HAL层直接操作寄存器

### 9.2 AB采样时序错位

- **现象**：低速时出现大量非法跳变
- **原因**：分两次读取A和B，中间发生状态变化
- **解决**：一次性读取整个端口，用位操作提取AB

### 9.3 临界区死锁

- **现象**：系统卡死
- **原因**：在临界区内调用其他可能阻塞的函数
- **解决**：临界区仅做"读取并清零"，其他逻辑放在临界区外

### 9.4 诊断计数器溢出

- **现象**：长时间运行后诊断数据异常
- **解决**：定期读取并清零诊断计数器，或使用32位变量（可支持约40亿次计数）

---

## 10. 总结

软件中断编码器的核心认知：

1. **本质**：用CPU时间替代硬件计数器，脉冲来源不变
2. **代价**：每次边沿触发20~50时钟周期的CPU开销
3. **边界**：适用于100kHz以下脉冲频率
4. **优势**：灵活诊断、不受TIM资源限制
5. **关键**：ISR必须极速，状态机查表法是最佳实践