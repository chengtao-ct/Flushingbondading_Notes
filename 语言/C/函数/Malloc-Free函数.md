## malloc和free函数的充分理解：

来源于Genimi大体写的在堆上分配内存的逻辑，其实里面更加复杂，而且每个操作系统都不一样

MMU 让软件工程师可以假装物理内存是完美的、连续的，而不需要关心底层的支离破碎

```c
#include <stdio.h>
#include <unistd.h> // for sbrk
// 内存块的元数据结构
typedef struct block_meta {
    size_t size;
    int is_free;
    struct block_meta *next_free; // 指向下一个空闲块
} block_meta;
// 全局变量，指向空闲链表的头部
void *free_list_head = NULL;
// 查找合适的空闲块
block_meta *find_free_block(size_t size) {
    block_meta *current = (block_meta*)free_list_head;
    while (current) {
        // 首次适应（First-Fit）算法：找到第一个足够大的块
        if (current->is_free && current->size >= size) {
            return current;
        }
        current = current->next_free;
    }
    return NULL; // 没找到
}
// 向操作系统申请新内存
block_meta *request_space(size_t size) {
    block_meta *block;
    // sbrk(0) 获取当前堆的末尾
    block = (block_meta*)sbrk(0); 
    // sbrk(size) 扩展堆的边界
    void *request = sbrk(size + sizeof(block_meta));
    if (request == (void*) -1) {
        return NULL; // 内存耗尽
    }
    block->size = size;
    block->is_free = 0;
    block->next_free = NULL;
    return block;
}
// --- malloc 的实现 ---
void *my_malloc(size_t size) {
    if (size <= 0) {
        return NULL;
    }

    block_meta *block;

    if (!free_list_head) { // 如果是第一次调用
        block = request_space(size);
        if (!block) {
            return NULL;
        }
        free_list_head = block;
    } else {
        // 1. 在空闲链表中查找合适的块
        block = find_free_block(size);
        if (block) { // 如果找到了
            block->is_free = 0; // 标记为“已使用”
                                // 实际实现中，还会进行分割（splitting）操作
                                // 即如果找到的块太大，会把它分割成两块
        } else { // 如果没找到
            // 2. 向操作系统申请一块新的内存
            block = request_space(size);
            if (!block) {
                return NULL;
            }
        }
    }
    
    // 3. 返回给用户的地址，是元数据之后的数据区起始地址
    return (block + 1);
}
// --- free 的实现 ---
void my_free(void *ptr) {
    if (!ptr) {
        return;
    }

    // 1. 通过用户指针，找到隐藏的元数据块的地址
    block_meta *block_ptr = (block_meta*)ptr - 1;

    // 2. 标记该块为空闲
    block_ptr->is_free = 1;

    // 3. (简化) 将其重新加入空闲链表的头部
    // 实际实现中，会在这里进行复杂的“合并”操作
    // 检查 block_ptr 的前一个和后一个物理相邻的块是否也为空闲
    // 如果是，就将它们合并成一个大的空闲块
    block_ptr->next_free = (block_meta*)free_list_head;
    free_list_head = block_ptr;
}
int main() {
    int *p = (int*)my_malloc(sizeof(int));
    if(p){
        *p = 100;
        printf("Value: %d\n", *p);
        my_free(p);
    }
    return 0;
}
```

这也能说明为什么使用堆的数据比使用栈的数据要慢和性能差，里面的调用是复杂的，内存储存是碎片化的，而且非常容易内存泄漏和野指针的情况，而栈上的数据符合先进后出的原则(从上到下)，是非常快捷的，但同时系统分配给栈的内存较少