---
aliases: [Finite State Machine, 有限状态机, 状态机, 数据驱动]
tags:
  - 嵌入式
  - 架构设计
  - FSM
  - 状态机
related:
  - "[[../04_FreeRTOS/任务管理/任务状态机]]"
  - "[[互斥与同步]]"
date: 2026-01-27
status: ✅已完成
chip: Generic
---

## 核心问题/概念
在嵌入式系统（如 STM32 智能小车底盘控制）中，彻底摒弃阻塞式的 `delay()` 和面条式的 `switch-case` 硬编码，采用**数据驱动、非阻塞、软硬分离**的有限状态机（FSM）架构，以应对复杂的业务逻辑和不可靠的物理硬件环境。

---

## 关键结论/解决方案

### 1. FSM 四大支柱
一切状态机均由以下四要素构成：

| 要素 | 说明 | 示例 |
|------|------|------|
| **现态** (Current State) | 当前所处的状态 | `STATE_IDLE` |
| **事件** (Event) | 触发状态改变的条件 | `EV_START_BTN` |
| **动作** (Action) | 状态转换时执行的操作 | `motor_start()` |
| **次态** (Next State) | 转换后的目标状态 | `STATE_RUNNING` |

> **关联**: [[状态机设计模式]] | [[事件驱动架构]]

---

### 2. 数据驱动逻辑（查表法）

将状态跳转规则封装为结构体数组（逻辑表），而非写死在代码流中。

#### 结构体定义
```c
// 状态与事件枚举
typedef enum {
    STATE_IDLE = 0,
    STATE_RUNNING,
    STATE_STOPPING,
    STATE_ERROR,
    STATE_MAX  // 边界检查用
} FSM_State;

typedef enum {
    EV_NONE = 0,
    EV_START_BTN,
    EV_STOP_BTN,
    EV_TIMEOUT,
    EV_ERROR,
    EV_MAX
} FSM_Event;

// 状态转移表项
typedef struct {
    FSM_State  curr_state;    // 现态
    FSM_Event  event;         // 触发事件
    void (*action)(void);     // 动作函数指针
    FSM_State  next_state;    // 次态
} FSM_Transition;
```

#### 状态转移表（示例：电机控制）
```c
// 前向声明动作函数
void act_motor_start(void);
void act_motor_stop(void);
void act_motor_brake(void);
void act_error_handler(void);
void act_reset_system(void);

// 状态转移表 - 数据驱动核心
const FSM_Transition transition_table[] = {
    // 现态              事件              动作函数            次态
    { STATE_IDLE,       EV_START_BTN,    act_motor_start,   STATE_RUNNING  },
    { STATE_IDLE,       EV_ERROR,        act_error_handler, STATE_ERROR    },
    { STATE_RUNNING,    EV_STOP_BTN,     act_motor_stop,    STATE_STOPPING },
    { STATE_RUNNING,    EV_TIMEOUT,      act_motor_brake,   STATE_ERROR    },
    { STATE_RUNNING,    EV_ERROR,        act_error_handler, STATE_ERROR    },
    { STATE_STOPPING,   EV_TIMEOUT,      act_reset_system,  STATE_IDLE     },
    { STATE_ERROR,      EV_START_BTN,    act_reset_system,  STATE_IDLE     },
};

#define TRANSITION_COUNT (sizeof(transition_table) / sizeof(FSM_Transition))
```

#### 通用 FSM 引擎（核心代码）
```c
typedef struct {
    FSM_State current_state;
    uint32_t  state_enter_time;  // 进入当前状态的时间戳
} FSM_Context;

static FSM_Context fsm = { STATE_IDLE, 0 };

// FSM 引擎 - 主循环中调用
void fsm_engine(FSM_Event event) {
    // 1. 状态非法检查（防御机制）
    if (fsm.current_state >= STATE_MAX) {
        fsm.current_state = STATE_ERROR;
        act_error_handler();
        return;
    }
    
    // 2. 遍历转移表
    for (uint8_t i = 0; i < TRANSITION_COUNT; i++) {
        const FSM_Transition *t = &transition_table[i];
        
        if (t->curr_state == fsm.current_state && t->event == event) {
            // 执行动作
            if (t->action != NULL) {
                t->action();
            }
            
            // 状态切换
            fsm.current_state = t->next_state;
            fsm.state_enter_time = get_system_tick();
            
            return;  // 匹配成功，退出
        }
    }
    // 未匹配项：可选择记录日志或忽略
}
```

> **关联**: [[查表法设计模式]] | [[函数指针实战应用]]

**感悟**(2026-3-6)：加了一个时间戳，是为了干啥，其实我看AI为什么比我写的好呢，对于边界的处理永远在考虑，这是我这个新手老是忘记的。还有就是他是靠遍历来选取的，感觉这是最死的方法，假如状态多了的化是不是有什么优化的算法。还有就是多用指针，少他妈生成内存。最搞笑的一开我看*，我他妈在想这个乘法是干啥的，太久不看代码了。

---

### 3. 核心防御机制

#### 3.1 状态非法检查
```c
// 方案一：枚举边界检查
if (fsm.current_state >= STATE_MAX) {
    fsm.current_state = STATE_IDLE;  // 强制拉回安全态
    hardware_emergency_stop();
}

// 方案二：状态有效性校验函数
bool is_valid_state(FSM_State s) {
    return (s >= STATE_IDLE && s < STATE_MAX);
}
```

#### 3.2 超时机制
将"超时"视为一种与外部输入平级的**事件**：

```c
// 状态超时配置表
typedef struct {
    FSM_State state;
    uint32_t  timeout_ms;  // 该状态的最大允许停留时间
} FSM_TimeoutConfig;

const FSM_TimeoutConfig timeout_table[] = {
    { STATE_IDLE,      0 },      // 0 表示无超时限制
    { STATE_RUNNING,   5000 },   // 运行态最多 5 秒
    { STATE_STOPPING,  1000 },   // 停止态最多 1 秒
};

// 在主循环中检测超时
void check_timeout(void) {
    uint32_t elapsed = get_system_tick() - fsm.state_enter_time;
    
    for (uint8_t i = 0; i < TIMEOUT_TABLE_SIZE; i++) {
        if (timeout_table[i].state == fsm.current_state) {
            if (timeout_table[i].timeout_ms > 0 && 
                elapsed >= timeout_table[i].timeout_ms) {
                fsm_engine(EV_TIMEOUT);  // 触发超时事件
            }
            break;
        }
    }
}
```
**感悟**：本质就是为了保证实时性
#### 3.3 临界区保护
```c
// 状态变量在中断中被修改时，必须加保护
void fsm_engine_safe(FSM_Event event) {
    __disable_irq();  // 进入临界区
    FSM_State cached_state = fsm.current_state;
    __enable_irq();   // 退出临界区
    
    // 使用 cached_state 进行后续处理...
}
```

> **关联**: [[嵌入式防御性编程]] | [[临界区与原子操作]]

---

### 4. 编码与桩测试

#### 软硬分离架构
```
┌─────────────────────────────────────┐
│           应用层 (FSM 逻辑)          │
├─────────────────────────────────────┤
│         抽象接口层 (函数指针)         │
├──────────────────┬──────────────────┤
│   桩实现 (PC测试) │  真实实现 (硬件)  │
│   - 键盘输入模拟  │  - HAL_GPIO_Read │
│   - printf 输出  │  - HAL_TIM_PWM   │
└──────────────────┴──────────────────┘
```

#### 桩函数示例
```c
#ifdef MOCK_MODE
// PC 测试环境 - 桩实现
void act_motor_start(void) {
    printf("[MOCK] Motor START\n");
}
void act_motor_stop(void) {
    printf("[MOCK] Motor STOP\n");
}
FSM_Event get_event_mock(void) {
    char c = getchar();
    switch (c) {
        case 's': return EV_START_BTN;
        case 'x': return EV_STOP_BTN;
        case 'e': return EV_ERROR;
        default:  return EV_NONE;
    }
}
#else
// 真实硬件环境
void act_motor_start(void) {
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
}
void act_motor_stop(void) {
    HAL_TIM_PWM_Stop(&htim3, TIM_CHANNEL_1);
}
#endif
```

> **关联**: [[单元测试框架]] | [[硬件抽象层设计]]

**感悟**：NB,都是为了进行软硬件分离，更好的进行debug，好像还可以用Python来做桩

---

## 避坑指南

| ❌ 错误做法 | ✅ 正确做法 | 原因 |
|-----------|-----------|------|
| 在 Action 中调用 `delay()` | 剥离为独立状态 + 超时机制 | 阻塞导致系统无响应 |
| 中断直接修改状态变量 | 使用临界区或原子操作 | 数据竞争风险 |
| 未经验证直接上机测试 | 先在 PC 端桩测试验证 | 防止硬件损坏 |
| 只设计 Happy Path | 增加 `STATE_ERROR` 容错态 | 异常情况必须有归宿 |
| 状态机无限膨胀 | 拆分为多个子状态机 | 单一职责原则 |

---

## 扩展：状态流转图示例

```
                    ┌──────────────────────────────────────┐
                    │                                      │
                    ▼                                      │
              ┌─────────┐                                  │
              │  IDLE   │◄─────────────────────────────────┤
              └────┬────┘                                  │
                   │ EV_START_BTN                          │
                   │ act: motor_start()                    │
                   ▼                                       │
              ┌─────────┐                                  │
         ┌───►│ RUNNING │◄───┐                             │
         │    └────┬────┘    │                             │
         │         │         │                             │
    EV_ERROR   EV_STOP_BTN  EV_START_BTN                   │
    act:error  act:stop()   (重复按下忽略)                  │
         │         │                                       │
         │         ▼                                       │
         │    ┌─────────┐                                  │
         │    │STOPPING │──EV_TIMEOUT──► IDLE              │
         │    └─────────┘     act:reset()                  │
         │                                                  │
         ▼                                                  │
    ┌─────────┐                                             │
    │  ERROR  │──────── EV_START_BTN ───────────────────────┘
    └─────────┘        act:reset()
```

---

## 标签与链接

### Tags
`#embedded/architecture` `#fsm` `#c-programming` `#debug/mock` `#rtos-prep` `#defensive-programming` `#state-pattern`

### 相关笔记链接
- [[状态机设计模式]] - UML 状态图与设计模式
- [[事件驱动架构]] - 消息队列与事件分发
- [[查表法设计模式]] - 数据驱动编程范式
- [[嵌入式防御性编程]] - EMI 防护与异常处理
- [[临界区与原子操作]] - 多线程/中断安全
- [[硬件抽象层设计]] - HAL 架构设计
- [[单元测试框架]] - Unity/CMock 实践
- [[RTOS 任务状态机]] - FreeRTOS 状态机集成
- [加了一个时间戳，是为了干啥，其实我看AI为什么比我写的好@20260306_175333](../../copilot/copilot-conversations/加了一个时间戳，是为了干啥，其实我看AI为什么比我写的好@20260306_175333.md)：

## 实战
- [[../工程实践/STM32小车-FSM设计]]
