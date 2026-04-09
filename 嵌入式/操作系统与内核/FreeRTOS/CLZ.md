## ARM CLZ 指令

**CLZ (Count Leading Zeros)** - 计算前导零指令

### 功能
统计 32 位寄存器中从最高位开始连续的零的个数。

### 示例
| 操作数 | 结果 |
|--------|------|
| `0x80000000` | 0 |
| `0x00000001` | 31 |
| `0x00010000` | 15 |
| `0x00000000` | 32 |

### 在 FreeRTOS 中的应用
用于快速查找最高优先级就绪任务：

```c
// 传统方法：循环遍历，O(n)
for (i = MAX_PRIORITY; i >= 0; i--) {
    if (readyList[i] != NULL) break;
}

// CLZ 优化：常数时间 O(1)
uint32_t readyBits = getReadyTaskBitmap();
int topPriority = 31 - __CLZ(readyBits);
```

### 优势
- 单周期指令执行
- 查找时间恒定，不受优先级数量影响
- 限制：最多支持 32 个优先级（32 位位图）

### 编译器内联函数
- **GCC**: `__builtin_clz(x)`
- **Keil**: `__clz(x)`
- **IAR**: `__CLZ(x)`