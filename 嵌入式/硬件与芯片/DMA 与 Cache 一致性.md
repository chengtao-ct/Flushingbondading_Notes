---
title: DMA 与 Cache 一致性
tags:
status: evergreen
created: 2026-03-03
updated: 2026-03-03
---

# DMA 与 Cache 一致性

## 一句话结论
在**非硬件一致性**系统中，[[CPU]] 主要读写 [[Cache]]，[[DMA]] 直接读写 [[RAM]]。如果不在所有权切换点做 cache 维护，就会出现随机错包、旧数据、偶现异常。

## 系统模型（Big Picture）
- [[CPU]]：面向 Cache 的高频访问方。
- [[DMA]]：外设与 RAM 之间的数据搬运方，不直接访问 CPU 私有 Cache。
- [[RAM]]：DMA 的真实读写目标。
- 核心矛盾：CPU 看到的“最新值”可能只在 Cache 中，DMA 看到的还是 RAM 旧值。

## 两个典型故障模型

### Tx 路径（CPU -> DMA -> 外设）
1. CPU 写发送 Buffer（数据先进入 Cache，未必立刻回写 RAM）。
2. DMA 从 RAM 读取，读到旧值。
3. 外设发出错误帧或陈旧数据。

### Rx 路径（外设 -> DMA -> CPU）
1. DMA 将新数据写入 RAM。
2. CPU 读取时命中旧 Cache Line。
3. CPU 看见旧数据，表现为“数据卡住”或“偶现不更新”。

## 根因与关键术语
- [[Write-Back]]：写回策略导致脏数据延迟落 RAM。
- [[Cache Line]]：cache 维护最小粒度是行，不是字节。
- [[False Sharing]]：DMA Buffer 与 CPU 热变量共享同一 Cache Line，易触发覆盖。
- [[Ownership Rule]]：map 给 DMA 后，CPU 在 unmap 前不得访问该区域。

## 必须遵守的工程规则
1. DMA Buffer 起始地址按 [[Cache Line]] 对齐。
2. DMA Buffer 长度按 Cache Line 整数倍对齐。
3. DMA Buffer 与 CPU 高频变量隔离，禁止共享一条 Cache Line。
4. 明确所有权边界：`map -> DMA owns -> unmap -> CPU owns`。
5. 禁止同时使用 cached/non-cached 别名地址访问同一物理区。

## 方案选型

### 方案 A：Coherent / Non-cacheable（稳定优先）
- 思路：通过 [[MMU]] / [[MPU]] 把共享区设为 Non-cacheable。
- Linux：`dma_alloc_coherent()`。
- FreeRTOS/裸机：将共享段配置为 `Non-cacheable + Shareable`。
- 适用：描述符、控制块、小规模高频共享区。
- 代价：CPU 访问性能下降。

### 方案 B：Streaming DMA（性能优先）
- 思路：数据面继续使用 cache，在边界点做 clean/invalidate。
- Linux 常见 API：`dma_map_single()` / `dma_unmap_single()`。
- FreeRTOS/OSAL 常见 API：`CacheP_wb()` / `CacheP_Inv()`。
- 适用：大块、一次性数据传输（网络 payload、音视频 buffer）。

## Linux 实战流程（简版）

### CPU -> 外设（`DMA_TO_DEVICE`）
1. CPU 填充 Buffer。
2. `dma_map_single(..., DMA_TO_DEVICE)`。
3. 启动 DMA。
4. 完成后 `dma_unmap_single()`。

### 外设 -> CPU（`DMA_FROM_DEVICE`）
1. 准备接收 Buffer。
2. `dma_map_single(..., DMA_FROM_DEVICE)`。
3. 启动 DMA 接收。
4. 完成后 `dma_unmap_single()`。
5. CPU 再读取数据。

## FreeRTOS / 裸机实战要点
- 若用 [[MPU]] 方案：在启动早期完成区域属性配置。
- 若用手动维护方案：
  - Tx 前 `CacheP_wb()`。
  - Rx 后 `CacheP_Inv()`。
- TI R5F 常见做法：覆盖弱符号配置（如 `gCslR5MpuCfg`），避免直接改原厂 `startup.c`。

## 高风险点：Cache Line 伪共享
示例：一个结构体中，`cpu_flag` 与 `dma_buf` 落在同一条 [[Cache Line]]。CPU 仅修改 `cpu_flag` 也会让整行变脏；随后错误的维护顺序可能把旧 `dma_buf` 回写覆盖 DMA 新数据。

建议：
- 控制字段与 DMA 数据区分离到不同 cache line。
- 对 descriptor/data 使用不同 section 与对齐策略。

## 排查清单（Debug Checklist）
- 是否确认平台是 coherent 还是 non-coherent DMA？
- DMA Buffer 是否 line 对齐且长度取整？
- map/unmap 生命周期内是否有 CPU 越权访问？
- 中断里是否在正确时机调用 clean/invalidate？
- 是否存在 cached/non-cached 混用别名？
- 描述符区与 payload 区是否已分策略管理？

## 常见误区
- 误区 1：出问题时只临时加一次 invalidate。
- 误区 2：把所有共享内存都设为 non-cacheable。
- 误区 3：直接修改 SDK 原厂启动文件，导致升级不可维护。


## 🔗 知识拓扑

- [多核并发的处理](../操作系统与内核/多核并发的处理.md)
- [DMA(直接存储器访问)](外设/DMA(直接存储器访问).md)