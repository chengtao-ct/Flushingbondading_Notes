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