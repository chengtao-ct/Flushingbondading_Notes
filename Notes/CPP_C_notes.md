# 关键字词与API
## static,extern,voliate的各自作用和互相关系(生命周期，跨文件声明和内存可见性)
1. static：储存在全局数据区，只可以被初始化一次。内部链接(只在当前文件可见)，在类里面，作为成员
2. extern:引用性声明,只声明，不初始化
3. voliate: 极易变动的数据，不要编译器的过度优化(不一定是代码修改，而是硬件，中断，线程修改)，
            volatile不能保证多线程环境下的原子性或操作顺序。在现代C++多线程编程中，你应该使用std::atomic和 std::mutex

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

## sleep()函数的理解

本质上sleep() 函数的执行，是一次典型的从用户态（User Mode）进入内核态（Kernel Mode），再返回用户态的过程

## _attribute_
 


## C语言里的voliate和Java里的voliate的区别



来源于Genimi

```c
// --- 这是在操作系统内核中运行的伪代码 ---

// 用户程序通过系统调用进入这里
/*
    sleep()挂起进程的作用
    参数一：进程
    参数二：时间
*/
void kernel_sleep(process_t* current_process, timespan_t duration) {

    // 1. 计算未来的唤醒时间
    time_t wakeup_time = get_current_time() + duration;

    // 2. 将当前进程的状态设为“睡眠”
    current_process->state = SLEEPING;
    current_process->wakeup_at = wakeup_time;

    // 3. 将其从就绪队列移除，放入等待队列
    remove_from_ready_queue(current_process);
    add_to_wait_queue(current_process); // 等待队列可能是按唤醒时间排序的

    // 4. (隐式) 调用调度器，让出CPU
    schedule_next_process(); 
}


// --- 这是硬件时钟中断触发的伪代码 ---
void on_timer_interrupt() {
    
    // 检查等待队列的头部
    while (wait_queue_is_not_empty() && wait_queue_top()->wakeup_at <= get_current_time())//一些带调度算法 {
        
        // 1. 时间到了，取出要唤醒的进程
        process_t* woken_process = remove_from_wait_queue();

        // 2. 改变其状态为“就绪”
        woken_process->state = READY;

        // 3. 将其重新放入就绪队列
        add_to_ready_queue(woken_process);
    }

    // ... 其他时钟中断处理 ...

```
总的来说分为这几个步骤：

-第一个函数申请进入内核模式，内核模式进行操作，比如设置定时器，队列的变换，调度器的调度，再会到用户模式

- 第二个函数进行硬件中断，先cpu声明时间到了，进行队列更新，等待调度


## open函数

## mlockall (全内存锁定)函数

# 指针操作

## 悬空指针：

指向已被释放的内存，会导致内存崩溃。

## 函数指针的理解和操作： 





# 线程和进程的一些基本操作(pthread,fork等)
> 进程有两种创建方式，一种是操作系统创建的，一种是父进程创建的。从计算机启动到终端执行程序的过程为：0号进程 -> 1号用户进程 -> 1号用户进程(init进程) -> getty进程 -> shell进程 -> 命令行执行进程。所以我们在命令行中通过./program执行文件时，所有创建的进程都是shell进程的子进程，这时候为什么shell一关闭，在shell中执行的进程都自动被关闭的原因。从shell进程到创建其他子进程需要通过以下接口.

**进程的操作(API)**：fork() ,exec(), wait(), exit(), getpid(), getppid()

**特点**： 
           1.子进程会复制父进程的所有资源，包括文件描述符、内存空间等。可以通过exec()函数加载新的程序代码。

           2.子进程与父进程之间是独立的，子进程的修改不会影响父进程。

           3.子进程通过exit(0)销毁自己。父进程通过wait(NULL)来等待并回收子进程的资源，这是非常重要的一步，可以防止“僵尸进程”的产生


> 来源于 github_阿秀

**线程的操作(API)**：

创建： int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg)->有点像cpp里面类的思想，面向对象：

来源于Genimi:

![]({C310FB8B-3A46-4071-A234-0E4F46DC523A}.png)

返回自身TID: pthread_t pthread_self()

等待线程结束：int pthread_join(pthread_t tid, void** retval)

![]({76661474-FF66-4E88-ADFB-271F56813922}.png)

结束线程：pthread_exit(void *retval);

分离线程：int pthread_detach(pthread_t tid);

**线程属性**：

```c
typedef struct{
    int etachstate;    // 线程分离的状态
    int schedpolicy;    // 线程调度策略
    struct sched_param schedparam;    // 线程的调度参数
    int inheritsched;    // 线程的继承性
    int scope;    // 线程的作用域
    // 以下为线程栈的设置
    size_t guardsize;    // 线程栈末尾警戒缓冲大小
    int stackaddr_set;    // 线程的栈设置
    void *    stackaddr;    // 线程栈的位置
    size_t stacksize;    // 线程栈大小
}pthread_arrt_t;

```


# C(编译性语言)的编译过程

[Core Dumped:How Real Projects Mix Compiled and Interpreted Languages](https://www.youtube.com/watch?v=RnBOOF502p0&t=229s)

还是推荐看这个Core Dumped的视频，这边深入的解释了不同语言的文件是怎么链接的(多语言程序)，这也与编译过程有关


来源：github.阿秀

四个过程：

**（1）预编译**
主要处理源代码文件中以“#”开头的预编译指令。处理规则见下

1、删除所有的#define，展开所有的宏定义。

2、处理所有的条件预编译指令，如“#if”、“#endif”、“#ifdef”、“#elif”和“#else”。

3、处理“#include”预编译指令，将文件内容替换到它的位置，这个过程是梯度进行的，文件中包含其他文件。

4、删除所有注释，“//”和“/**/”。

5、保留所有的#pragma编译器指令，编译器需要使用它们，如：#pragma Once是为了防止有文件被重引用。

6、添加行号和文件标识，然后编译时编译器产生调试用的行号信息，并且编译时产生编译错误或警告是能够显示行号。

**（2）编译**
把预编译之后生成的xxx.i或xxx.ii文件，进行一系列词法分析、语法分析、语义分析及优化后，生成相应的汇编代码文件。

1、词法分析：利用类似于“有限状态机”的算法，将源代码程序输入到扫描机中，将其中的字符序列分割成一系列的记号。

2、语法分析：语法分析器对由扫描器产生的记号，进行语法分析，产生语法树。由语法分析器输出的语法树是一种以表达式为节点的树。

3、语义分析：语法分析器只是完成了对表达式语法层面的分析，语义分析器则对表达式是否有意义进行判断，其分析的语义是静态语义——在编译期能分期的语义，相对应的动态语义是在运行期才能确定的语义。

4、优化：源代码级别的一个优化过程。

5、目标代码生成：由代码生成器将中间代码转换成目标机器代码，生成一系列的代码序列——汇编语言表示。

6、目标代码优化：目标代码优化器对上述的目标机器代码进行优化：寻找合适的寻址方式、使用位移来替代乘法运算、删除多余的指令等。

**（3）汇编**

将汇编代码转变成机器可以执行的指令(机器码文件)。 汇编器的汇编过程相对于编译器来说更简单，没有复杂的语法，也没有语义，更不需要做指令优化，只是根据汇编指令和机器指令的对照表一一翻译过来，汇编过程有汇编器as完成。

经汇编之后，产生目标文件(与可执行文件格式几乎一样)xxx.o(Linux下)、xxx.obj(Windows下)。

**（4）链接**

将不同的源文件产生的目标文件进行链接，从而形成一个可以执行的程序。链接分为静态链接和动态链接：

**1、静态链接：**
函数和数据被编译进一个二进制文件。在使用静态库的情况下，在编译链接可执行文件时，链接器从库中复制这些函数和数据并把它们和应用程序的其它模块组合起来创建最终的可执行文件。
空间浪费：因为每个可执行程序中对所有需要的目标文件都要有一份副本，所以如果多个程序对同一个目标文件都有依赖，会出现同一个目标文件都在内存存在多个副本；
更新困难：每当库函数的代码修改了，这个时候就需要重新进行编译链接形成可执行程序。

运行速度快：但是静态链接的优点就是，在可执行程序中已经具备了所有执行程序所需要的任何东西，在执行的时候运行速度快。

**2、动态链接：**
动态链接的基本思想是把程序按照模块拆分成各个相对独立部分，在程序运行时才将它们链接在一起形成一个完整的程序，而不是像静态链接一样把所有程序模块都链接成一个单独的可执行文件。

共享库：就是即使需要每个程序都依赖同一个库，但是该库不会像静态链接那样在内存中存在多份副本，而是这多个程序在执行时共享同一份副本；

更新方便：更新时只需要替换原来的目标文件，而无需将所有的程序再重新链接一遍。当程序下一次运行时，新版本的目标文件会被自动加载到内存并且链接起来，程序就完成了升级的目标。

性能损耗：因为把链接推迟到了程序运行时，所以每次执行程序都需要进行链接，所以性能会有一定损失。









           

