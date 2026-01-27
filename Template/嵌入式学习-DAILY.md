<%*
// 1. 定义变量 (保持不变)
const direction = await tp.system.prompt("核心方向? (如: rtos, linux, bare-metal)", "bare-metal")
const chip = await tp.system.prompt("硬件平台/芯片型号? (如: STM32, ESP32)", "Generic")
const module = await tp.system.prompt("涉及外设/模块? (如: I2C, SPI, DMA)", "None")
const lang = await tp.system.prompt("编程语言? (C/C++/Rust/Assembly)", "C")
const title = tp.file.title
const date = tp.date.now("YYYY-MM-DD HH:mm")
-%>
---
created: <% date %>
updated: <% date %>
tags:
  - embedded
  - <% direction %>
chip: <% chip %>
module: <% module %>
language: <% lang %>
status: 🟢进行中
---

# 📑 <% title %>

> [!abstract] 学习目标
> 本次学习主要为了理解什么概念？解决什么疑惑？
> <% tp.file.cursor() %>

## 🧠 核心原理 (Theory)
<!-- 纯文字理论：用自己的话解释“这是什么”以及“为什么需要它” -->
- **定义/概念**: 
- **工作机制**: 
- **关键公式/参数**: 

## 📊 图表与架构分析 (Diagrams)
<!-- 核心区域：粘贴 Datasheet 里的【系统框图】、【时序图】或【状态机图】 -->

> [!example] 读图分析
> **(在这里写你对上图的理解)**
> *例如：观察时序图，数据是在 SCL 的上升沿被采样，起始信号是 SCL 为高时 SDA 拉低...*


## 💻 代码逻辑与实现 (<% lang %>)

```<% lang.toLowerCase() %>
// 这里的代码侧重于“演示原理”，而非“工程驱动”
// 比如：展示结构体的定义，或者关键寄存器的位操作宏