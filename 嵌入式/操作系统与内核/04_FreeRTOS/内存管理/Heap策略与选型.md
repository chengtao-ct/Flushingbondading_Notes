---
aliases: [Heap策略, heap_1 heap_2 heap_3 heap_4 heap_5, FreeRTOS堆选型]
tags:
  - 嵌入式
  - FreeRTOS
  - RTOS
  - 内存管理
  - 堆策略
related:
  - "[[内存管理_概览]]"
  - "[[异构内存与任意堆]]"
  - "[[OOM与MallocFailedHook]]"
date: 2026-03-03
status: ✅已完成
chip: Generic
---


> [!abstract] 核心本质
> FreeRTOS提供5种Heap方案，本质是**时间确定性、内存利用率、碎片化**三者的权衡。`heap_4`是通用最优解（首次适应+相邻合并），`heap_5`专攻异构内存，选型错误是嵌入式内存问题的根源。

## 一、核心定义

**Heap策略**：动态内存分配的底层实现机制，决定：
- 分配算法：如何找到合适大小的空闲块
- 碎片处理：释放后如何合并相邻空闲块
- 时间确定性：分配/释放的时间复杂度是否可预测

**核心痛点**：
- 碎片化：总空闲内存足够，但无法分配大块连续内存
- 时间抖动：分配时间不确定，影响实时性
- 内存浪费：内部碎片（分配块大于请求）+ 外部碎片（空闲块分散）

---

## 二、底层原理

### 2.1 五种Heap方案全景对比

| 特性 | heap_1 | heap_2 | heap_3 | heap_4 | heap_5 |
|------|--------|--------|--------|--------|--------|
| **分配能力** | 只分配 | 分配+释放 | 分配+释放 | 分配+释放 | 分配+释放 |
| **释放能力** | ❌ 无 | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| **碎片处理** | 无碎片 | 不合并 | 依赖标准库 | **相邻合并** | **相邻合并** |
| **分配算法** | 简单递增 | 最佳匹配 | 标准malloc | **首次适应** | **首次适应** |
| **时间确定性** | ✅ O(1) | ⚠️ O(n) | ❌ 不确定 | ⚠️ O(n) | ⚠️ O(n) |
| **多内存区** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **适用场景** | 静态系统 | 固定大小块 | 简单原型 | **通用主力** | **异构SoC** |

### 2.2 heap_4核心数据结构

```c
// heap_4.c - 块链接结构
typedef struct A_BLOCK_LINK
{
    struct A_BLOCK_LINK *pxNextFreeBlock;  // 下一个空闲块指针
    size_t xBlockSize;                      // 块大小（含头部）
} BlockLink_t;

// 关键常量
#define heapBLOCK_SIZE_IS_VALID( xBlockSize )    ( ( xBlockSize ) & heapBLOCK_ALLOCATED_BITMASK )
#define heapBLOCK_ALLOCATED_BITMASK              ( ( size_t ) 0x80000000 )  // 最高位标记已分配
#define heapMINIMUM_BLOCK_SIZE                   ( ( size_t ) ( xHeapStructSize << 1 ) )

// 全局空闲链表（按地址排序！）
static BlockLink_t xStart;
static BlockLink_t *pxEnd = NULL;

// 堆内存区域
static uint8_t ucHeap[configTOTAL_HEAP_SIZE];
```

#### 内存块布局

```
已分配块：
┌────────────────────────────────────────────────┐
│ BlockLink_t (8字节)                             │ ← pxNextFreeBlock = NULL (或特殊标记)
│ xBlockSize | 0x80000000                         │ ← 最高位置1表示已分配
├────────────────────────────────────────────────┤
│                                                │
│           用户数据区                            │ ← pvPortMalloc返回值
│           (请求大小 + 对齐填充)                  │
│                                                │
└────────────────────────────────────────────────┘

空闲块：
┌────────────────────────────────────────────────┐
│ BlockLink_t (8字节)                             │ ← pxNextFreeBlock指向下一个空闲块
│ xBlockSize (最高位为0)                          │
├────────────────────────────────────────────────┤
│                                                │
│           未使用空间                            │
│                                                │
└────────────────────────────────────────────────┘
```

### 2.3 heap_4分配算法（首次适应）

```c
// heap_4.c - pvPortMalloc 核心逻辑（简化）
void *pvPortMalloc(size_t xWantedSize)
{
    BlockLink_t *pxBlock, *pxPreviousBlock, *pxNewBlockLink;
    void *pvReturn = NULL;
    
    vTaskSuspendAll();  // 挂起调度器，保证原子性
    
    // 1. 首次调用时初始化堆
    if (pxEnd == NULL) {
        prvHeapInit();
    }
    
    // 2. 调整请求大小（对齐 + 头部开销）
    xWantedSize += xHeapStructSize;
    xWantedSize = (xWantedSize + portBYTE_ALIGNMENT_MASK) & ~portBYTE_ALIGNMENT_MASK;
    
    // 3. 遍历空闲链表，找第一个足够大的块（首次适应）
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;
    
    while ((pxBlock->xBlockSize < xWantedSize) && (pxBlock->pxNextFreeBlock != NULL)) {
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock->pxNextFreeBlock;
    }
    
    // 4. 找到合适块
    if (pxBlock != pxEnd) {
        pvReturn = (void *)(((uint8_t *)pxBlock) + xHeapStructSize);
        
        // 5. 如果块远大于需求，分裂
        if ((pxBlock->xBlockSize - xWantedSize) > heapMINIMUM_BLOCK_SIZE) {
            pxNewBlockLink = (BlockLink_t *)(((uint8_t *)pxBlock) + xWantedSize);
            pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;
            pxBlock->xBlockSize = xWantedSize;
            
            // 插入空闲链表
            pxNewBlockLink->pxNextFreeBlock = pxBlock->pxNextFreeBlock;
            pxBlock->pxNextFreeBlock = pxNewBlockLink;
        }
        
        // 6. 从空闲链表移除
        pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock;
        
        // 7. 标记为已分配
        pxBlock->xBlockSize |= heapBLOCK_ALLOCATED_BITMASK;
        pxBlock->pxNextFreeBlock = NULL;
    }
    
    xTaskResumeAll();
    return pvReturn;
}
```

### 2.4 heap_4释放与合并算法

```c
// heap_4.c - vPortFree 核心逻辑（简化）
void vPortFree(void *pv)
{
    BlockLink_t *pxLink;
    
    if (pv == NULL) return;
    
    // 1. 获取块头部
    pxLink = (BlockLink_t *)(((uint8_t *)pv) - xHeapStructSize);
    
    // 2. 清除分配标记
    pxLink->xBlockSize &= ~heapBLOCK_ALLOCATED_BITMASK;
    
    vTaskSuspendAll();
    
    // 3. 按地址插入空闲链表（关键：保持地址有序）
    BlockLink_t *pxIterator = &xStart;
    while (pxIterator->pxNextFreeBlock < pxLink) {
        pxIterator = pxIterator->pxNextFreeBlock;
    }
    pxLink->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
    pxIterator->pxNextFreeBlock = pxLink;
    
    // 4. 尝试与下一个块合并
    if (((uint8_t *)pxLink) + pxLink->xBlockSize == (uint8_t *)pxLink->pxNextFreeBlock) {
        pxLink->xBlockSize += pxLink->pxNextFreeBlock->xBlockSize;
        pxLink->pxNextFreeBlock = pxLink->pxNextFreeBlock->pxNextFreeBlock;
    }
    
    // 5. 尝试与前一个块合并
    if (((uint8_t *)pxIterator) + pxIterator->xBlockSize == (uint8_t *)pxLink) {
        pxIterator->xBlockSize += pxLink->xBlockSize;
        pxIterator->pxNextFreeBlock = pxLink->pxNextFreeBlock;
    }
    
    xTaskResumeAll();
}
```

> [!tip] 关键洞察
> 空闲链表**按地址排序**是heap_4能合并相邻块的关键！heap_2按大小排序，无法合并，因此碎片化严重。

### 2.5 碎片化原理

```
初始状态（10KB堆）：
[========== 空闲 ==========]

分配 A=2KB, B=2KB, C=2KB：
[AA][BB][CC][==== 空闲 ====]

释放 B：
[AA][  ][CC][==== 空闲 ====]
      ↑ 2KB空洞

分配 D=3KB：
[AA][  ][CC][DDDDDDD][ 空闲 ]
      ↑ 2KB空洞无法利用！

碎片化：总空闲 = 2KB + 3KB = 5KB，但无法分配3KB连续块
```

**heap_4的解决方案**：
```
释放 A 后：
[  ][  ][CC][==== 空闲 ====]
   ↑   ↑
   合并！heap_4检测到相邻空闲块，自动合并为4KB

释放 C 后：
[========== 空闲 ==========]
   全部合并！恢复完整10KB
```

---

## 三、实战应用落地

### 3.1 选型决策树

```
                    ┌─────────────────────┐
                    │ 是否需要动态释放？   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │ 否                               │ 是
              ▼                                  ▼
      ┌───────────────┐              ┌─────────────────────┐
      │   heap_1      │              │ 是否有多个非连续     │
      │ (最简单确定)   │              │ 内存区域？           │
      └───────────────┘              └──────────┬──────────┘
                                                │
                               ┌────────────────┴────────────────┐
                               │ 是                               │ 否
                               ▼                                  ▼
                       ┌───────────────┐              ┌─────────────────────┐
                       │   heap_5      │              │ 分配大小是否固定？   │
                       │ (异构内存)     │              └──────────┬──────────┘
                       └───────────────┘                         │
                                                    ┌────────────┴────────────┐
                                                    │ 是                       │ 否
                                                    ▼                          ▼
                                            ┌───────────────┐        ┌───────────────┐
                                            │   heap_2      │        │   heap_4      │
                                            │ (固定大小块)   │        │ (通用最优解)   │
                                            └───────────────┘        └───────────────┘
```

### 3.2 配置示例

```c
// FreeRTOSConfig.h - Heap配置

// 方式1：使用heap_4（推荐）
#define configAPPLICATION_ALLOCATED_HEAP   0
// 在项目中只编译 heap_4.c

// 方式2：自定义堆位置（heap_4/heap_5）
#define configAPPLICATION_ALLOCATED_HEAP   1
// 需要在代码中定义：
// uint8_t ucHeap[configTOTAL_HEAP_SIZE] __attribute__((section(".sram_heap")));

// 方式3：heap_5多区域配置
// 见 [[异构内存与任意堆]]

// 堆大小配置
#define configTOTAL_HEAP_SIZE              ((size_t) (40 * 1024))  // 40KB

// 内存分配失败Hook
#define configUSE_MALLOC_FAILED_HOOK       1
```

### 3.3 内存监控代码

```c
// memory_monitor.c

#include "FreeRTOS.h"
#include "task.h"

// 碎片化指标计算
typedef struct {
    size_t totalHeap;           // 总堆大小
    size_t freeHeap;            // 当前空闲
    size_t minEverFree;         // 历史最小空闲
    size_t largestFreeBlock;    // 最大连续空闲块
    uint8_t fragmentation;      // 碎片化程度 (0-100%)
} HeapStats_t;

void vGetHeapStats(HeapStats_t *pxStats)
{
    pxStats->totalHeap = configTOTAL_HEAP_SIZE;
    pxStats->freeHeap = xPortGetFreeHeapSize();
    pxStats->minEverFree = xPortGetMinimumEverFreeHeapSize();
    
    // 计算最大连续空闲块（需要遍历空闲链表）
    // heap_4没有直接API，需要修改源码或估算
    // 简化估算：假设碎片化程度
    pxStats->largestFreeBlock = pxStats->freeHeap;  // 最乐观估计
    
    // 碎片化程度 = 1 - (最大块 / 总空闲)
    if (pxStats->freeHeap > 0) {
        pxStats->fragmentation = 
            (uint8_t)((1.0 - (float)pxStats->largestFreeBlock / pxStats->freeHeap) * 100);
    }
}

// 定期监控任务
void vHeapMonitorTask(void *pvParameters)
{
    HeapStats_t xStats;
    const TickType_t xInterval = pdMS_TO_TICKS(10000);  // 10秒
    
    for (;;) {
        vGetHeapStats(&xStats);
        
        printf("\n=== Heap Monitor ===\n");
        printf("Total:     %u bytes\n", xStats.totalHeap);
        printf("Free:      %u bytes (%.1f%%)\n", 
               xStats.freeHeap, 
               (float)xStats.freeHeap / xStats.totalHeap * 100);
        printf("Min Ever:  %u bytes\n", xStats.minEverFree);
        printf("Fragment:  %u%%\n", xStats.fragmentation);
        
        // 预警
        if (xStats.freeHeap < xStats.totalHeap / 4) {
            printf("⚠️ WARNING: Heap < 25%%\n");
        }
        if (xStats.minEverFree < xStats.totalHeap / 10) {
            printf("🚨 CRITICAL: Historical minimum < 10%%\n");
        }
        
        vTaskDelay(xInterval);
    }
}
```

### 3.4 内存池替代方案

```c
// memory_pool.h - 避免碎片的内存池设计

typedef struct {
    uint8_t *pucPoolMemory;
    size_t xBlockSize;
    size_t xBlockCount;
    size_t xAllocatedCount;
    uint8_t *pucFreeList;  // 空闲块链表头
} MemoryPool_t;

// 创建固定大小内存池
MemoryPool_t *pxMemoryPoolCreate(size_t xBlockSize, size_t xBlockCount)
{
    MemoryPool_t *pxPool;
    uint8_t *pucMemory;
    
    // 分配池结构
    pxPool = pvPortMalloc(sizeof(MemoryPool_t));
    if (pxPool == NULL) return NULL;
    
    // 分配池内存
    pucMemory = pvPortMalloc(xBlockSize * xBlockCount);
    if (pucMemory == NULL) {
        vPortFree(pxPool);
        return NULL;
    }
    
    pxPool->pucPoolMemory = pucMemory;
    pxPool->xBlockSize = xBlockSize;
    pxPool->xBlockCount = xBlockCount;
    pxPool->xAllocatedCount = 0;
    
    // 构建空闲链表
    pxPool->pucFreeList = pucMemory;
    for (size_t i = 0; i < xBlockCount - 1; i++) {
        uint8_t *pucCurrent = pucMemory + i * xBlockSize;
        uint8_t *pNext = pucCurrent + xBlockSize;
        *(uint8_t **)pucCurrent = pNext;  // 指向下一块
    }
    *(uint8_t **)(pucMemory + (xBlockCount - 1) * xBlockSize) = NULL;  // 最后一块
    
    return pxPool;
}

// O(1) 分配
void *pvMemoryPoolAlloc(MemoryPool_t *pxPool)
{
    if (pxPool->pucFreeList == NULL) return NULL;  // 池耗尽
    
    void *pvBlock = pxPool->pucFreeList;
    pxPool->pucFreeList = *(uint8_t **)pvBlock;  // 取链表头
    pxPool->xAllocatedCount++;
    
    return pvBlock;
}

// O(1) 释放
void vMemoryPoolFree(MemoryPool_t *pxPool, void *pvBlock)
{
    *(uint8_t **)pvBlock = pxPool->pucFreeList;  // 插入链表头
    pxPool->pucFreeList = (uint8_t *)pvBlock;
    pxPool->xAllocatedCount--;
}
```

---

## 四、避坑与边界条件

> [!danger] 致命陷阱
> 1. **heap_2用于可变大小分配**：碎片无法合并，长时间运行后必然失败
> 2. **heap_3与标准库冲突**：如果其他代码也用 `malloc()`，堆管理会混乱
> 3. **只看 `xPortGetFreeHeapSize()`**：忽略碎片化，大块分配可能失败
> 4. **heap_5区域表未排序**：地址乱序会导致内存破坏

> [!warning] 避坑指南
> - 优先使用 `heap_4`，除非有明确理由选其他
> - 高频分配的固定大小对象，使用内存池而非通用堆
> - 长生命周期对象（任务栈、队列）优先静态分配
> - 监控 `xPortGetMinimumEverFreeHeapSize()` 而非当前空闲

### 4.1 常见错误模式

```c
// ❌ 错误1：频繁分配释放不同大小
void vBadPattern(void)
{
    for (int i = 0; i < 100; i++) {
        void *p1 = pvPortMalloc(100 + i * 10);  // 不同大小
        void *p2 = pvPortMalloc(200);
        // ... 使用 ...
        vPortFree(p1);
        vPortFree(p2);
    }
    // 结果：严重碎片化
}

// ✅ 正确：使用内存池
MemoryPool_t *pxPool = pxMemoryPoolCreate(256, 50);

void vGoodPattern(void)
{
    void *p1 = pvMemoryPoolAlloc(pxPool);  // O(1)，无碎片
    void *p2 = pvMemoryPoolAlloc(pxPool);
    // ... 使用 ...
    vMemoryPoolFree(pxPool, p1);
    vMemoryPoolFree(pxPool, p2);
}
```

### 4.2 验证清单

- [ ] 是否明确当前工程使用的 `heap_x.c`？
- [ ] 是否有对象生命周期表（创建/销毁时机）？
- [ ] 是否持续监控 `xPortGetMinimumEverFreeHeapSize()`？
- [ ] 是否存在"总内存够但大块申请失败"的情况？
- [ ] 高频动态对象是否已迁移到内存池？
- [ ] heap_5的区域表是否按地址升序排列？

---

## 🔗 知识延伸

- ⬆️ **上位知识**：[[内存管理_概览]]、[[动态内存分配原理]]
- ⬇️ **下位知识**：[[异构内存与任意堆]]、[[OOM与MallocFailedHook]]、[[Memory Pool设计模式]]
- ➡️ **平级关联**：[[../任务管理/动态分配生命周期与Idle回收]]、[[../任务管理/任务栈与溢出防护]]

---

> [!note] 修改说明
> 本次修改补全了五种Heap方案的全景对比表、heap_4核心数据结构与分配/释放算法源码分析、碎片化原理图解、选型决策树、内存监控代码、内存池替代方案，并建立了与异构内存、OOM处理的知识关联。
