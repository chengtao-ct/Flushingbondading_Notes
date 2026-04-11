---
aliases: [Memory Management Unit, 内存管理单元, 虚拟内存, 页表]
tags: [硬件架构, 内存管理, ARM, Linux内核, 嵌入式开发]
date: 2026-03-03
status: 🌿草稿
---

> [!abstract] 核心本质
> MMU是CPU与总线之间的硬件模块，实现**虚拟地址到物理地址的翻译**。它让每个进程拥有独立的虚拟地址空间，实现进程隔离、内存保护、按需分配三大核心能力。理解MMU是从"裸机思维"跨越到"操作系统思维"的关键门槛。

## 一、核心定义

### 1.1 什么是MMU？

**MMU（Memory Management Unit）**：现代处理器中的硬件模块，位于CPU内核与总线/Cache之间，负责：
- 地址翻译：虚拟地址（VA）→ 物理地址（PA）
- 访问权限检查：读/写/执行权限、特权级验证
- 内存属性管理：Cache策略、共享属性

### 1.2 为什么需要MMU？

**裸机时代的问题**：

```
物理内存布局（无MMU）：
┌─────────────────────────────────────────────────────┐ 0x0000_0000
│  程序 A 代码区                                       │
├─────────────────────────────────────────────────────┤
│  程序 A 数据区                                       │
├─────────────────────────────────────────────────────┤ 0x2000_0000
│  程序 B 代码区                                       │
├─────────────────────────────────────────────────────┤
│  程序 B 数据区                                       │
└─────────────────────────────────────────────────────┘

问题：程序 B 的野指针写入 0x0000_0000 → 直接破坏程序 A → 系统崩溃
```

**引入MMU后**：

```
虚拟地址空间（每个进程独立）：
进程 A 视角：                    进程 B 视角：
┌─────────────────┐             ┌─────────────────┐
│ 0x0000_0000     │             │ 0x0000_0000     │
│    代码区        │             │    代码区        │
├─────────────────┤             ├─────────────────┤
│    数据区        │             │    数据区        │
├─────────────────┤             ├─────────────────┤
│    堆栈区        │             │    堆栈区        │
└─────────────────┘             └─────────────────┘
        ↓ MMU 翻译                      ↓ MMU 翻译
物理内存（实际布局）：
┌─────────────────────────────────────────────────────┐
│  进程 A 物理页（分散在各个位置）                      │
├─────────────────────────────────────────────────────┤
│  进程 B 物理页（分散在各个位置）                      │
└─────────────────────────────────────────────────────┘

优势：进程 B 写 0x0000_0000 → MMU 检查权限 → 触发异常 → 保护进程 A
```

### 1.3 三大核心能力

| 能力 | 解决的问题 | 实现机制 |
|------|-----------|----------|
| **地址重映射** | 物理内存碎片化、进程隔离 | 虚拟地址连续 → 物理地址分散 |
| **内存保护** | 非法访问、权限越界 | 页表项中的权限位检查 |
| **按需分配** | 内存浪费、过度承诺 | 缺页异常触发物理页分配 |

---

## 二、底层原理

### 2.1 核心类比

> [!tip] 邮局类比
> - **CPU** = 寄信人，写的是虚拟地址（"张三收"）
> - **MMU** = 邮局地址转译中心，查阅"密码本"
> - **页表** = 密码本，记录"张三"→"南京路88号"的映射
> - **物理内存** = 真实街道地址

### 2.2 页与页框

MMU按**页**为单位进行翻译，而非字节：

| 概念 | 定义 | 大小 |
|------|------|------|
| **页** | 虚拟内存的分配单位 | 通常 4KB |
| **页框** | 物理内存的分配单位 | 与页相同，4KB |
| **页表项（PTE）** | 记录一个页到页框的映射 | 8字节（64位系统） |

```
虚拟地址空间                    物理内存
┌──────┐ Page 0                ┌──────┐ Frame 0
│      │ ─────────────────────→ │      │
├──────┤                        ├──────┤ Frame 1
│      │ Page 1                 │      │
├──────┤                        ├──────┤ Frame 2
│      │ Page 2 ──────────────→ │      │
├──────┤                        ├──────┤
│      │ Page 3                 │      │
└──────┘                        └──────┘

关键：虚拟页可以映射到任意物理页框，实现"逻辑连续、物理分散"
```

### 2.3 多级页表

**问题**：64位地址空间极大，单级页表会耗尽内存。

```
假设：虚拟地址 48位，页大小 4KB
单级页表大小 = 2^48 / 4KB * 8字节 = 512 GB  ← 不可接受！
```

**解决方案**：多级页表（ARMv8 通常 4 级）

```
虚拟地址结构（48位）：
┌────────────┬────────────┬────────────┬────────────┬─────────────┐
│ PGD索引(9) │ PUD索引(9) │ PMD索引(9) │ PTE索引(9) │ 页内偏移(12)│
└────────────┴────────────┴────────────┴────────────┴─────────────┘
     ↓              ↓           ↓           ↓            ↓
  查 PGD         查 PUD      查 PMD      查 PTE       物理地址

每级页表大小 = 2^9 * 8 = 4KB（刚好一页！）

查表过程：
1. 从 TTBR 寄存器获取 PGD 基址
2. 用 PGD 索引查 PGD，得到 PUD 基址
3. 用 PUD 索引查 PUD，得到 PMD 基址
4. 用 PMD 索引查 PMD，得到 PTE 基址
5. 用 PTE 索引查 PTE，得到物理页框号
6. 物理页框号 + 页内偏移 = 物理地址
```

### 2.4 TLB（转换旁路缓存）

**问题**：每次访存都要查 4 级页表 = 4 次内存访问，性能不可接受。

**解决方案**：TLB 缓存最近的翻译结果

```
MMU 地址翻译流程：

CPU 发出虚拟地址
        │
        ▼
┌───────────────┐
│  TLB 查找     │
└───────┬───────┘
        │
   ┌────┴────┐
   │         │
 Hit        Miss
   │         │
   ▼         ▼
直接返回   触发 Page Table Walk
物理地址         │
                 ▼
         ┌───────────────┐
         │ 遍历 4 级页表  │
         │ (硬件自动完成) │
         └───────┬───────┘
                 │
                 ▼
         ┌───────────────┐
         │ 更新 TLB      │
         └───────┬───────┘
                 │
                 ▼
           返回物理地址

性能对比：
- TLB Hit：1 周期
- TLB Miss：几十到上百周期（取决于内存延迟）
```

### 2.5 ARMv8 关键寄存器

| 寄存器 | 全称 | 作用 |
|--------|------|------|
| **TTBR0_EL1** | Translation Table Base Register 0 | 用户态页表基址 |
| **TTBR1_EL1** | Translation Table Base Register 1 | 内核态页表基址 |
| **TCR_EL1** | Translation Control Register | 页表属性配置（粒度、地址宽度等） |
| **SCTLR_EL1** | System Control Register | MMU 使能位（M bit） |

> [!info] 进程切换的本质
> Linux 进程切换时，只需将 `TTBR0` 改为新进程的 PGD 物理地址，整个用户态虚拟空间瞬间切换！

---

## 三、实战应用落地

### 3.1 系统启动流程中的MMU

```
启动阶段              MMU 状态           地址模式
─────────────────────────────────────────────────
BootROM              关闭               PA = VA
    │
    ▼
U-Boot 早期          关闭               PA = VA
    │
    ▼
U-Boot 后期          开启               Flat Mapping
    │                                  (VA = PA)
    ▼
Linux Kernel 入口    开启               早期页表
(head.S)                                (临时映射)
    │
    ▼
paging_init()        开启               完整内核映射
    │
    ▼
用户态程序运行        开启               每进程独立页表
```

### 3.2 缺页异常处理流程

```c
// 用户程序
void *ptr = malloc(1024 * 1024);  // 申请 1MB
memset(ptr, 0, 1024 * 1024);      // 触发缺页异常

/*
 * 内核处理流程：
 * 
 * 1. malloc() 系统调用
 *    → 内核在进程 VMA 中预留 1MB 虚拟地址范围
 *    → 但 PTE 为空，未映射物理页
 * 
 * 2. memset() 首次访问
 *    → MMU 查表失败
 *    → 触发 Data Abort 异常
 * 
 * 3. Linux 异常处理程序
 *    → do_page_fault()
 *    → 检查 VMA 合法性
 *    → 合法：调用 alloc_pages() 分配物理页
 *    → 更新 PTE，建立映射
 * 
 * 4. 返回用户态
 *    → 重新执行 memset() 指令
 *    → MMU 查表成功，写入数据
 */
```

### 3.3 内核中常用的地址转换API

```c
#include <linux/mm.h>
#include <linux/dma-mapping.h>

// 虚拟地址 → 物理地址（仅适用于直接映射区）
phys_addr_t virt_to_phys(volatile void *address);

// 物理地址 → 虚拟地址（仅适用于低端内存）
void *phys_to_virt(phys_addr_t address);

// 用户空间虚拟地址 → 物理页
struct page *get_user_pages(vma, start, nr_pages, flags, pages);

// DMA 一致性内存分配（自动处理 Cache 属性）
void *dma_alloc_coherent(struct device *dev, size_t size, 
                         dma_addr_t *dma_handle, gfp_t flag);

// DMA 流式映射（需要手动维护 Cache）
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr, 
                          size_t size, enum dma_data_direction direction);
void dma_unmap_single(struct device *dev, dma_addr_t dma_handle, 
                      size_t size, enum dma_data_direction direction);
```

### 3.4 查看进程页表

```bash
# 查看进程的内存映射
cat /proc/<pid>/maps

# 示例输出
# 地址范围              权限  偏移     设备   inode   路径
# 00400000-0040b000     r-xp  00000000 08:01 12345   /usr/bin/ls
# 0060a000-0060b000     r--p  0000a000 08:01 12345   /usr/bin/ls
# 0060b000-0060c000     rw-p  0000b000 08:01 12345   /usr/bin/ls
# 7f8c40000000-7f8c40210000 rw-p 00000000 00:00 0    [heap]

# 查看页表详细信息（需要 root）
cat /proc/<pid>/pagemap
```

---

## 四、避坑与边界条件

> [!danger] 致命陷阱一：DMA 与 Cache 一致性
> 
> **现象**：CPU 构造数据 → DMA 发送 → 外设收到乱码
> 
> **根因**：CPU 写的数据还在 Cache 中，DMA 直接访问 DDR 拿到旧数据
> 
> **解决方案**：
> ```c
> // 方案1：使用一致性内存（推荐）
> void *buf = dma_alloc_coherent(dev, size, &dma_addr, GFP_KERNEL);
> // 内核会配置该页为 Non-cacheable
> 
> // 方案2：流式 DMA + 手动 Cache 维护
> dma_addr_t dma_addr = dma_map_single(dev, buf, size, DMA_TO_DEVICE);
> // 内核会自动 Flush Cache
> // DMA 传输完成后
> dma_unmap_single(dev, dma_addr, size, DMA_TO_DEVICE);
> ```

> [!danger] 致命陷阱二：TLB 陈旧数据
> 
> **现象**：修改页表权限后，CPU 仍然报权限错误
> 
> **根因**：TLB 缓存了旧的翻译结果
> 
> **解决方案**：
> ```c
> // 修改页表后必须刷新 TLB
> #include <asm/tlbflush.h>
> 
> // 刷新单个页
> flush_tlb_page(vma, address);
> 
> // 刷新整个地址空间
> flush_tlb_mm(mm);
> 
> // 刷新所有（慎用）
> flush_tlb_all();
> ```

> [!warning] 避坑指南三：用户空间与内核空间地址
> 
> - 用户空间地址（0x0000_0000 ~ 0x0000_7FFF_FFFF_FFFF）使用 `TTBR0`
> - 内核空间地址（0xFFFF_0000_0000_0000 ~ 0xFFFF_FFFF_FFFF_FFFF）使用 `TTBR1`
> - 在内核中直接解引用用户空间指针是**危险操作**，必须使用 `copy_from_user()` / `copy_to_user()`

### 4.1 常见错误模式

```c
// ❌ 错误1：内核中直接访问用户空间地址
ssize_t my_read(struct file *filp, char __user *buf, 
                size_t count, loff_t *ppos)
{
    // 危险！用户空间页面可能未映射或被换出
    // memcpy(kernel_buf, buf, count);  // 可能崩溃
    
    // 正确做法
    if (copy_from_user(kernel_buf, buf, count))
        return -EFAULT;
}

// ❌ 错误2：DMA 使用栈内存
void my_dma_transfer(void)
{
    char buf[1024];  // 栈上分配
    // 危险！栈内存物理不连续，且 Cache 行为不可控
    // dma_addr = dma_map_single(dev, buf, 1024, DMA_TO_DEVICE);
    
    // 正确做法：使用 kmalloc 或 dma_alloc_coherent
    void *buf = kmalloc(1024, GFP_KERNEL);
}

// ❌ 错误3：忘记 unmap
void bad_dma(void)
{
    dma_addr_t dma = dma_map_single(dev, buf, size, DMA_TO_DEVICE);
    // 使用 DMA...
    // 忘记 dma_unmap_single() → 资源泄漏，Cache 不一致
}
```

### 4.2 调试技巧

```bash
# 查看 TLB 信息（ARM64）
cat /sys/kernel/debug/tlb/tlb_stats

# 查看内存碎片化程度
cat /proc/buddyinfo

# 查看进程内存使用
pmap -x <pid>

# 追踪缺页异常
perf record -e page-faults -p <pid>
perf report
```

---

## 🔗 知识延伸

- ⬆️ **上位知识**：[[计算机体系结构]]、[[操作系统原理]]
- ⬇️ **下位知识**：[[Cache一致性协议]]、[[DMA与Cache一致性]]、[[Linux内存管理]]、[[ARMv8架构手册]]
- ➡️ **平级关联**：[[../操作系统与内核/FreeRTOS/内存管理/内存管理_概览]]（FreeRTOS视角）、[[虚拟化技术]]、[[进程地址空间]]

---

> [!note] 修改说明
> 本次修改补全了YAML前言、核心摘要、多级页表详细结构、TLB工作流程、ARMv8关键寄存器、缺页异常处理流程、内核API示例、常见错误模式、调试技巧，并建立了与FreeRTOS内存管理、DMA/Cache一致性的知识关联。