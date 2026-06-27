---
date: 2026-04-17
---
# ESP32 内存架构全景解析

## 一、先建立整体概念：三种存储器的角色

```mermaid
graph TB
    subgraph ESP32芯片存储架构
        A[内部 ROM<br/>448KB<br/>只读/断电不丢<br/>芯片内部/出厂固化]
        B[内部 SRAM<br/>520KB<br/>可读写/断电丢失<br/>芯片内部/运行时用]
        C[外部 Flash<br/>4MB<br/>可读写/断电不丢<br/>芯片外部/板载焊接]
        D[外部 PSRAM<br/>4MB<br/>可读写/断电丢失<br/>芯片外部/板载焊接]
    end
```

---

## 二、4MB Flash — 存放什么？

Flash 是 ESP32 的 *"硬盘"*，断电不丢失，所有持久化数据都在这里：

### 4MB Flash 物理地址空间（`0x000000` ~ `0x3FFFFF`）

| 偏移地址 | 大小 | 内容 | 作用 |
|---|---|---|---|
| `0x000000` | 4KB | MBT | Master Boot Record（出厂已写）<br/>存储 Flash 启动模式、频率等参数 |
| `0x001000` | 16~20KB | Bootloader | 二级引导程序（你自己编译烧录的）<br/>负责读取分区表、加载 APP、验证签名 |
| `0x008000` | 3KB | Partition Table | 分区表<br/>告诉 Bootloader 各区域在哪、多大 |
| `0x010000`<br/>（`0x110000`） | 1.8~3MB | APP 固件 | 你写的应用程序（最大的区域）<br/>包含代码段（`.text`）+ 只读数据（`.rodata`）<br/>+ FreeRTOS + Wi-Fi 协议栈 + 你的代码 |
| `0x310000`<br/>（示例） | 1MB | SPIFFS/<br/>NVS/FatFS | 文件系统区域<br/>存网页文件、配置文件、OTA 数据 |
| `0x9000` | 16KB | NVS | Non-Volatile Storage<br/>键值对存储：Wi-Fi 密码、校准数据 |
| `0x4000` | 8KB | PHY 数据 | Wi-Fi/蓝牙 校准参数 |

### 各区域详细说明

#### ① `0x000000` — MBT（Master Boot Table）

这块出厂时 Espressif 已经烧好了，你一般不用管。

**内容：**

```mermaid
graph LR
    A[MBT] --> B[Flash 访问模式<br/>DIO/DOUT/QIO/QOUT]
    A --> C[Flash 频率<br/>40MHz/80MHz]
    A --> D[芯片型号标识]
    A --> E[引导日志<br/>Boot log]
```

**作用：** 芯片上电后，ROM Bootloader 先读这里，知道 Flash 怎么访问、从哪加载 Bootloader。

#### ② `0x001000` — Bootloader（二级引导）

你的项目编译后生成的 `bootloader.bin`

**作用：**

1. 初始化 SPI Flash（切换到高速模式）
2. 读取 `0x8000` 处的分区表
3. 根据分区表找到 APP 分区
4. 验证 APP 的 SHA256/签名（如果启用了 Secure Boot）
5. 将 APP 的 `.text` 段映射到内存
6. 跳转到 APP 的入口地址执行

**为什么需要它？**

ROM Bootloader 太简单，只能加载一个固定地址。有了二级 Bootloader 才能支持：

- 多分区（OTA A/B 切换）
- Flash 加密
- 安全启动
- 自定义分区布局

#### ③ `0x008000` — 分区表

分区表本身就是一个 *"目录索引"*：

| # | Name | Type | SubType | Offset | Size |
|---|---|---|---|---|---|
| 1 | nvs | data | nvs | `0x9000` | 16KB |
| 2 | phy_init | data | phy | `0xf000` | 4KB |
| 3 | factory | app | factory | `0x10000` | 3MB |
| 4 | storage | data | spiffs | `0x310000` | 1MB |

**作用：** Bootloader 读这张表，才知道：

- APP 固件在哪个偏移地址
- NVS 数据区在哪
- 文件系统区在哪
- 有没有 OTA 分区（A 区/B 区）

#### ④ `0x010000` — APP 固件

你的应用程序，编译后的 `.bin` 文件，通常 1~3MB

**内部结构：**

```mermaid
graph TB
    A[APP 固件] --> B[.text<br/>可执行代码<br/>你的代码 + FreeRTOS + Wi-Fi 协议栈 + 驱动]
    A --> C[.rodata<br/>只读常量数据<br/>字符串常量、查表、摄像头驱动参数]
    A --> D[.data<br/>已初始化的全局/静态变量<br/>启动时从 Flash 复制到 SRAM]
    A --> E[.bss<br/>未初始化的全局/静态变量<br/>启动时在 SRAM 中清零<br/>本身不占 Flash 空间，只记录大小]
```

#### ⑤ `0x9000` — NVS（Non-Volatile Storage）

键值对存储，类似 Android 的 SharedPreferences

**典型存储内容：**

```mermaid
graph LR
    A[NVS] --> B[Wi-Fi SSID + 密码]
    A --> C[蓝牙配对信息]
    A --> D[PHY 校准数据<br/>每块芯片出厂不同！]
    A --> E[OTA 分区状态<br/>标记当前从哪个分区启动]
    A --> F[用户自定义配置<br/>设备 ID、服务器地址等]
    A --> G[Boot count<br/>启动次数计数]
```

**特点：**

- 通过 `nvs_set_str` / `nvs_get_str` 读写
- 内部使用磨损均衡（Flash 有擦写寿命）
- 即使 OTA 升级，NVS 数据不丢

---

## 三、520KB SRAM — 存放什么？

SRAM 是 ESP32 的 *"运行内存"*，速度极快但断电丢失：

### 520KB 内部 SRAM 布局

```mermaid
graph TB
    subgraph 内部SRAM布局
        A[Free Heap 动态内存堆<br/>约 320KB<br/>malloc/free<br/>任务栈<br/>摄像头小帧缓冲<br/>Wi-Fi 接收缓冲<br/>你的动态分配数据]
        B[.data 段<br/>约 100KB<br/>已初始化全局变量<br/>例: int count = 42<br/>启动时从 Flash 复制]
        C[.bss 段<br/>约 100KB<br/>未初始化全局变量<br/>例: static int buffer8192<br/>启动时自动清零]
    end
    
    D[⚠️ 实际可用堆 ≈ 200~250KB<br/>其余被系统占用]
```

### SRAM 里的典型消耗者

| 组件 | 占用 |
|---|---|
| Wi-Fi 协议栈 | ~50KB |
| FreeRTOS | ~10KB（内核 + IDLE 任务 + Timer 任务） |
| TCP/IP 栈 (lwIP) | ~40KB |
| 蓝牙（如果开启） | ~60KB |
| 摄像头驱动 | ~20KB |
| 你的代码栈+堆 | 剩余部分 |

总需求轻松超过 300KB！所以 ESP32-CAM 必须用 PSRAM 补充

---

## 四、448KB ROM — 存放什么？

ROM 是芯片出厂时 **光刻写入** 的，永远不可修改：

### 448KB 内部 ROM 布局

```mermaid
graph TB
    subgraph 内部ROM布局
        A[一级 Bootloader<br/>芯片上电第一条执行的代码<br/>读取 Flash MBT<br/>加载 0x1000 处 Bootloader]
        B[基础加密算法库<br/>AES 加解密<br/>SHA-256 / SHA-512<br/>RSA 验证<br/>ECC 椭圆曲线<br/>随机数生成器]
        C[基础外设驱动<br/>UART 最小驱动<br/>SPI Flash 读取<br/>GPIO 基础操作]
        D[芯片校准数据<br/>每颗芯片唯一<br/>MAC 地址 Wi-Fi/BT<br/>ADC 校准曲线<br/>Flash 电压参数]
    end
    
    E[🔒 只读！你永远无法写入或修改 ROM]
```

### ROM 的特殊性

**为什么需要 ROM？**

- 没有 ROM，芯片上电不知道该干什么（Flash 里什么都还没烧）
- ROM 保证任何状态下都能通过 UART 恢复（救砖）
- 加密算法固化在 ROM 里 → Secure Boot 无法被篡改

### ROM vs Flash vs SRAM 对比

| | ROM | Flash | SRAM | PSRAM |
|---|---|---|---|---|
| 位置 | 芯片内部 | 芯片外部 | 芯片内部 | 芯片外部 |
| 可写 | ❌ | ✅ | ✅ | ✅ |
| 速度 | 快 | 慢（需映射） | 最快 | 较慢 |
| 断电 | 保留 | 保留 | 丢失 | 丢失 |
| 容量 | 448KB | 4MB | 520KB | 4MB |
| 存什么 | 固化代码 | 固件+数据 | 运行时 | 大缓冲区 |

---

## 五、4MB PSRAM — 存放什么？

PSRAM（Pseudo-SRAM，伪静态 RAM）本质是一块通过 SPI 接口连接的外部 RAM

### 4MB PSRAM 布局

```mermaid
graph TB
    subgraph PSRAM布局
        A[摄像头帧缓冲<br/>最大消耗者]
        B[双帧缓冲<br/>fb1 = 3.75MB UXGA<br/>fb2 = 3.75MB<br/>放不下！UXGA 实际用 JPEG 模式压缩<br/>VGA 直出 = 640×480×2 = 600KB ✅ 可以]
        C[JPEG 编码输出缓冲<br/>UXGA JPEG 一张 ≈ 40~100KB]
        D[图像处理临时缓冲]
        E[大块动态分配<br/>溢出到 PSRAM 的 malloc<br/>大数组、JSON 解析缓冲<br/>HTTP 响应缓冲<br/>WebSocket 发送队列]
    end
    
    F[⚠️ 速度比内部 SRAM 慢 ~10 倍]
    G[⚠️ DMA 不能直接访问 PSRAM<br/>部分操作有限制]
    H[⚠️ 中断处理函数不要用 PSRAM 里的数据]
```

---

## 六、完整启动流程中各存储器的协作

```mermaid
flowchart TB
    A[上电瞬间] --> B[① CPU 从 ROM 地址 0x40000000 开始执行<br/>ROM Bootloader]
    B --> C[② ROM 读取 Flash 0x000000 处的 MBT<br/>得知 Flash 模式/频率/芯片型号]
    C --> D[③ ROM 加载 Flash 0x001000 处的 Bootloader 到 SRAM<br/>跳转到 SRAM 中的 Bootloader 执行]
    D --> E[④ Bootloader 初始化 PSRAM<br/>如果配置了]
    E --> F[⑤ Bootloader 读取 Flash 0x008000 处的分区表<br/>知道 APP 在 0x010000]
    F --> G[⑥ Bootloader 将 APP 的 .text 映射到 CPU 地址空间<br/>Flash APP → CPU 通过 Cache 直接执行 XIP 方式<br/>.data 段从 Flash 复制到 SRAM<br/>.bss 段在 SRAM 中清零]
    G --> H[⑦ 启动 FreeRTOS<br/>SRAM 中运行<br/>创建 idle 任务、timer 任务]
    H --> I[⑧ 调用 app_main<br/>你的代码开始执行！]
```

---

## 七、一句话总结

| 存储器 | 一句话 |
|---|---|
| ROM | 出厂固化，芯片上电第一条指令的来源，永远改不了 |
| Flash | 外部 *"硬盘"*，存固件、分区表、NVS、文件系统，断电不丢 |
| SRAM | 内部 *"内存"*，跑代码的运行时数据，断电即丢，非常紧张 |
| PSRAM | 外部 *"扩展内存"*，专门补 SRAM 不够用的场景（摄像头帧缓冲） |