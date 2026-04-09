---
title: OOM 防线与 Malloc Failed Hook
aliases:
  - vApplicationMallocFailedHook
  - 内存耗尽兜底
tags:
  - freertos
  - memory
  - reliability
status: evergreen
created: 2026-03-03
updated: 2026-03-03
source:
  - "[[内存管理_概览_原文归档_2026-03-03]]"
related:
  - "[[内存管理_概览]]"
  - "[[内存管理_动态分配生命周期与Idle回收]]"
---
---


> [!abstract] 核心本质
> `vApplicationMallocFailedHook()` 是FreeRTOS内存分配失败的**最后一道防线**。它的职责是**受控失效**——记录现场、安全复位，而非尝试在线修复。OOM后的系统状态已不可信，任何复杂操作都可能雪上加霜。

## 一、核心定义

**OOM（Out of Memory）**：堆内存耗尽，`pvPortMalloc()` 返回 `NULL`。

**Malloc Failed Hook**：FreeRTOS在内存分配失败时调用的用户回调函数，用于：
- 记录故障现场（任务名、时间戳、剩余内存）
- 触发系统复位或进入安全状态
- 避免系统在不可预测状态下继续运行

**核心痛点**：
- OOM后继续运行 → 空指针解引用 → HardFault
- 故障现场丢失 → 无法定位根因
- Hook中做复杂操作 → 二次崩溃

---

## 二、底层原理

### 2.1 Hook调用链源码分析

```c
// heap_4.c - pvPortMalloc 失败时的处理
void *pvPortMalloc(size_t xWantedSize)
{
    BlockLink_t *pxBlock;
    void *pvReturn = NULL;
    
    vTaskSuspendAll();
    {
        // ... 分配逻辑 ...
        
        // 如果遍历完空闲链表仍未找到合适块
        if (pvReturn == NULL) {
            // 关键：调用用户Hook
            #if (configUSE_MALLOC_FAILED_HOOK == 1)
            {
                extern void vApplicationMallocFailedHook(void);
                vApplicationMallocFailedHook();
            }
            #endif
        }
    }
    xTaskResumeAll();
    
    return pvReturn;  // 返回 NULL
}
```

> [!warning] 关键认知
> Hook在 `vTaskSuspendAll()` 保护下调用，**调度器已被挂起**！此时：
> - 不能调用 `vTaskDelay()`、`xQueueSend()` 等阻塞API
> - 不能调用 `pvPortMalloc()`（会死锁）
> - 中断仍然可以响应

### 2.2 调用上下文分析

| 调用场景 | 调度器状态 | 中断状态 | 可安全操作 |
|----------|------------|----------|------------|
| 任务中调用 `pvPortMalloc()` | 已挂起 | 可响应 | 写全局变量、GPIO、简单日志 |
| 中断中调用 `pvPortMalloc()` | 未挂起 | 当前中断 | 更受限，避免嵌套 |
| 启动前调用（调度器未启动） | 无效 | 可响应 | 初始化阶段，直接复位 |

### 2.3 OOM根因分类

```
OOM 根因分析
├── 真泄漏
│   ├── 忘记释放
│   ├── 释放失败（条件分支遗漏）
│   └── 循环引用（链表/树结构）
│
├── 峰值超预算
│   ├── 并发请求叠加
│   ├── 大缓冲区临时申请
│   └── 任务栈动态增长
│
└── 回收滞后
    ├── Idle饥饿（见 [[内存管理_动态分配生命周期与Idle回收]]）
    └── 碎片化导致大块申请失败
```

### 2.4 与HardFault的关系

```
OOM 后的可能路径：

路径1：检查返回值 → 正确处理 → 安全降级
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ pvPortMalloc│ ──→ │ 返回 NULL   │ ──→ │ 业务降级    │
│ 失败        │     │ 检查并处理   │     │ 安全运行    │
└─────────────┘     └─────────────┘     └─────────────┘

路径2：忽略返回值 → 空指针解引用 → HardFault
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ pvPortMalloc│ ──→ │ 返回 NULL   │ ──→ │ 解引用 NULL │
│ 失败        │     │ 未检查      │     → HardFault! │
└─────────────┘     └─────────────┘     └─────────────┘

路径3：Hook兜底 → 记录现场 → 受控复位
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ pvPortMalloc│ ──→ │ Hook 触发   │ ──→ │ 记录+复位   │
│ 失败        │     │             │     │ 可复盘      │
└─────────────┘     └─────────────┘     └─────────────┘
```

---

## 三、实战应用落地

### 3.1 基础配置

```c
// FreeRTOSConfig.h
#define configUSE_MALLOC_FAILED_HOOK       1
#define configCHECK_FOR_STACK_OVERFLOW     2
#define configRECORD_STACK_HIGH_ADDRESS    1

// 如果使用静态分配，还需要：
#define configSUPPORT_STATIC_ALLOCATION    1
#define configSUPPORT_DYNAMIC_ALLOCATION   1
```

### 3.2 故障记录结构设计

```c
// fault_record.h

typedef struct {
    uint32_t magic;                    // 魔数校验：0xDEADBEEF
    uint32_t faultType;                // 故障类型枚举
    char taskName[configMAX_TASK_NAME_LEN];
    uint32_t tick;                     // 故障发生时刻
    size_t freeHeap;                   // 当前空闲堆
    size_t minEverFree;                // 历史最小空闲堆
    size_t requestedSize;              // 请求分配的大小
    uint32_t checksum;                 // 校验和
} FaultRecord_t;

// 故障类型枚举
typedef enum {
    FAULT_TYPE_OOM = 1,
    FAULT_TYPE_STACK_OVERFLOW = 2,
    FAULT_TYPE_HARDFAULT = 3,
    FAULT_TYPE_ASSERT = 4,
} FaultType_e;

// 全局故障记录（放在保留RAM区，复位后不丢失）
__attribute__((section(".noinit"))) 
static FaultRecord_t gFaultRecord;

// 或使用备份寄存器（STM32）
// #define FAULT_RECORD_BKP  RTC_BKP_DR0
```

### 3.3 Hook实现

```c
// fault_handler.c

#include "FreeRTOS.h"
#include "task.h"

// 外部声明（链接脚本定义的保留区）
extern uint32_t __fault_record_start__;
extern uint32_t __fault_record_end__;

// 计算校验和
static uint32_t prvCalculateChecksum(FaultRecord_t *pxRecord)
{
    uint32_t sum = 0;
    uint8_t *pData = (uint8_t *)pxRecord;
    
    // 跳过 checksum 字段本身
    for (size_t i = 0; i < offsetof(FaultRecord_t, checksum); i++) {
        sum += pData[i];
    }
    return sum ^ 0xFFFFFFFF;
}

// Malloc Failed Hook 实现
void vApplicationMallocFailedHook(void)
{
    // ⚠️ 此时调度器已挂起，只能做最简单的操作！
    
    // 1. 记录故障信息
    gFaultRecord.magic = 0xDEADBEEF;
    gFaultRecord.faultType = FAULT_TYPE_OOM;
    
    // 获取当前任务名（安全：不涉及调度）
    TaskHandle_t xTask = xTaskGetCurrentTaskHandle();
    if (xTask != NULL) {
        strncpy(gFaultRecord.taskName, 
                pcTaskGetName(xTask), 
                configMAX_TASK_NAME_LEN - 1);
    } else {
        strcpy(gFaultRecord.taskName, "ISR/Init");
    }
    
    gFaultRecord.tick = xTaskGetTickCount();
    gFaultRecord.freeHeap = xPortGetFreeHeapSize();
    gFaultRecord.minEverFree = xPortGetMinimumEverFreeHeapSize();
    
    // 请求大小无法直接获取，记录为0
    gFaultRecord.requestedSize = 0;
    
    // 计算校验和
    gFaultRecord.checksum = prvCalculateChecksum(&gFaultRecord);
    
    // 2. 翻转错误指示LED（硬件操作，安全）
    #ifdef ERR_LED_GPIO
    GPIO_WriteBit(ERR_LED_GPIO, ERR_LED_PIN, Bit_SET);
    #endif
    
    // 3. 等待日志刷新（如果有UART FIFO）
    // for (volatile int i = 0; i < 100000; i++);  // 简单延时
    
    // 4. 触发复位
    NVIC_SystemReset();
    
    // 永不到达
    while (1);
}

// Stack Overflow Hook 实现（关联）
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    // 类似处理，记录不同故障类型
    gFaultRecord.magic = 0xDEADBEEF;
    gFaultRecord.faultType = FAULT_TYPE_STACK_OVERFLOW;
    strncpy(gFaultRecord.taskName, pcTaskName, configMAX_TASK_NAME_LEN - 1);
    gFaultRecord.tick = xTaskGetTickCount();
    gFaultRecord.freeHeap = xPortGetFreeHeapSize();
    gFaultRecord.minEverFree = xPortGetMinimumEverFreeHeapSize();
    gFaultRecord.checksum = prvCalculateChecksum(&gFaultRecord);
    
    NVIC_SystemReset();
}
```

### 3.4 启动时故障复盘

```c
// main.c - 系统启动时检查上次故障

void vCheckPreviousFault(void)
{
    // 检查魔数和校验和
    if (gFaultRecord.magic == 0xDEADBEEF) {
        uint32_t expectedChecksum = prvCalculateChecksum(&gFaultRecord);
        
        if (gFaultRecord.checksum == expectedChecksum) {
            // 有效故障记录，输出诊断信息
            printf("\n");
            printf("╔══════════════════════════════════════╗\n");
            printf("║     ⚠️  PREVIOUS FAULT DETECTED       ║\n");
            printf("╠══════════════════════════════════════╣\n");
            
            const char *faultTypeStr[] = {
                [FAULT_TYPE_OOM] = "Out of Memory",
                [FAULT_TYPE_STACK_OVERFLOW] = "Stack Overflow",
                [FAULT_TYPE_HARDFAULT] = "HardFault",
                [FAULT_TYPE_ASSERT] = "Assertion Failed",
            };
            
            printf("║ Type:     %-26s ║\n", faultTypeStr[gFaultRecord.faultType]);
            printf("║ Task:     %-26s ║\n", gFaultRecord.taskName);
            printf("║ Tick:     %-26lu ║\n", gFaultRecord.tick);
            printf("║ FreeHeap: %-26u ║\n", gFaultRecord.freeHeap);
            printf("║ MinFree:  %-26u ║\n", gFaultRecord.minEverFree);
            printf("╚══════════════════════════════════════╝\n\n");
            
            // 根据故障类型给出建议
            if (gFaultRecord.faultType == FAULT_TYPE_OOM) {
                if (gFaultRecord.minEverFree < 100) {
                    printf("→ 建议：检查内存泄漏，minEverFree 接近0\n");
                } else {
                    printf("→ 建议：增大 configTOTAL_HEAP_SIZE 或优化峰值使用\n");
                }
            }
        }
        
        // 清除记录，避免下次误判
        memset(&gFaultRecord, 0, sizeof(gFaultRecord));
    }
}

int main(void)
{
    // 硬件初始化
    HAL_Init();
    SystemClock_Config();
    
    // 检查上次故障
    vCheckPreviousFault();
    
    // 创建任务...
    xTaskCreate(vMainTask, "Main", 256, NULL, 1, NULL);
    
    vTaskStartScheduler();
    
    while (1);
}
```

### 3.5 OOM测试与演练

```c
// test_oom.c - 主动构造OOM场景验证Hook

void vTestOomTask(void *pvParameters)
{
    printf("=== OOM Test Start ===\n");
    
    size_t freeHeap = xPortGetFreeHeapSize();
    printf("Free heap: %u bytes\n", freeHeap);
    
    // 逐步分配直到耗尽
    void *ptrs[100];
    int count = 0;
    
    for (int i = 0; i < 100; i++) {
        ptrs[i] = pvPortMalloc(1024);  // 每次申请1KB
        if (ptrs[i] == NULL) {
            printf("OOM at allocation #%d (total: %d KB)\n", 
                   i, i * 1024 / 1024);
            break;
        }
        count++;
    }
    
    // 如果到这里，说明Hook没有触发复位（配置错误？）
    printf("⚠️ Hook did not trigger reset!\n");
    
    // 释放测试内存
    for (int i = 0; i < count; i++) {
        vPortFree(ptrs[i]);
    }
    
    vTaskDelete(NULL);
}

// 在测试任务中调用
void vRunOomTest(void)
{
    xTaskCreate(vTestOomTask, "OOM_Test", 128, NULL, 2, NULL);
}
```

---

## 四、避坑与边界条件

> [!danger] 致命陷阱
> 1. **Hook中调用阻塞API**：`vTaskDelay()`、`xQueueSend()` 会死锁或崩溃
> 2. **Hook中再次分配内存**：`pvPortMalloc()` 会死锁
> 3. **Hook中做复杂日志输出**：`printf()` 可能需要动态内存或中断，不可靠
> 4. **忽略返回值继续运行**：空指针解引用导致HardFault，比OOM更难排查

> [!warning] 避坑指南
> - Hook中只做：写全局变量、操作GPIO、触发复位
> - 故障记录放在 `.noinit` 区或备份寄存器，确保复位后可读
> - 区分"泄漏"和"峰值超预算"：看 `minEverFree` 是否持续下降
> - 定期演练OOM场景，验证Hook和复位路径

### 4.1 泄漏 vs 峰值超预算判断

```c
// 在监控任务中定期记录
void vMemoryMonitorTask(void *pvParameters)
{
    static size_t xLastMinEverFree = 0xFFFFFFFF;
    
    for (;;) {
        size_t xCurrentMinFree = xPortGetMinimumEverFreeHeapSize();
        
        // 判断趋势
        if (xCurrentMinFree < xLastMinEverFree) {
            printf("⚠️ MinFree下降: %u -> %u (可能泄漏)\n", 
                   xLastMinEverFree, xCurrentMinFree);
            xLastMinEverFree = xCurrentMinFree;
        }
        
        vTaskDelay(pdMS_TO_TICKS(60000));  // 每分钟检查
    }
}

// OOM发生时的判断逻辑：
// 如果 minEverFree 接近0 → 真泄漏或长期回收滞后
// 如果 minEverFree 还有较大空间 → 峰值超预算或碎片化
```

### 4.2 验证清单

- [ ] `configUSE_MALLOC_FAILED_HOOK` 是否设为1？
- [ ] Hook函数是否已实现并链接正确？
- [ ] 故障记录是否放在复位保留区？
- [ ] 启动时是否检查并输出上次故障信息？
- [ ] 是否定期演练OOM场景？
- [ ] 所有 `pvPortMalloc()` 调用是否都检查了返回值？
- [ ] 是否能区分"泄漏"和"峰值超预算"？

---

## 🔗 知识延伸

- ⬆️ **上位知识**：[[内存管理_概览]]、[[嵌入式系统可靠性设计]]
- ⬇️ **下位知识**：[[内存管理_Heap策略与选型]]、[[HardFault排查手册]]、[[看门狗设计]]
- ➡️ **平级关联**：[[内存管理_动态分配生命周期与Idle回收]]、[[内存管理_任务栈与溢出防护]]

---

> [!note] 修改说明
> 本次修改补全了Hook调用链源码分析、调用上下文分析、OOM根因分类、故障记录结构设计、完整的Hook实现代码、启动时故障复盘机制、OOM测试演练代码，并建立了与HardFault、Idle回收的知识关联。