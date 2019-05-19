# LAB7 实验报告

## 练习1

**1. 对比一下练习0之后的代码和lab6有什么不同.**

对比命令如下

```c
diff -bur --suppress-common-lines --suppress-blank-empty . ../lab6
```

下面分析一下diff出来的不同(只分析大的方面, 小的地方还有注释中指明的就不说了)

- ./kern/mm/vmm.c & ./kern/mm/vmm.h
    - mm_struct 里面增加了mm_sem 和 locked_by 数据, mm_lock数据没了
    - lock_init 改成了 sem_init, 为了配合信号量的实现吧
    - lock_mm 和 unlock_mm 里都加入了相关的逻辑
- ./kern/process/proc.c & ./kern/process/proc.h
    - 新增了 do_sleep 函数的实现
    - 为了哲学家就餐问题, 在 init_main 里调了一下 check_sync 函数
    - proc.h 里面加了几个新的宏定义
- ./kern/schedule/sched.c & ./kern/schedule/sched.h
    - 加入了 add_timer, del_timer, run_timer_list 的实现
    - 加入了 timer_t 数据结构和 timer_init 初始化函数
- ./kern/sync/sync.h
    - 删了几个和lock有关的函数, 因为新的版本是用的信号量
- ./kern/syscall/syscall.c
    - 新增了 sys_sleep 中断, 调用的是前面新加的 do_sleep 函数
- ./lib/atomic.h
    - 删了 test_and_set_bit 和 test_and_clear_bit 函数
- ./user 文件夹下的更改基本上都是关于sys_sleep的, 和kern的更改对应, 就不说了
- 新增文件
    - ./kern/sync: check_sync.c
    - ./kern/sync: monitor.c
    - ./kern/sync: monitor.h
    - ./kern/sync: sem.c
    - ./kern/sync: sem.h
    - ./kern/sync: wait.c
    - ./kern/sync: wait.h


**2. 请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程。**

信号量的数据结构, 以及相关的数据结构

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;

typedef struct {
    list_entry_t wait_head;
} wait_queue_t;

typedef struct {
    struct proc_struct *proc;
    uint32_t wakeup_flags;
    wait_queue_t *wait_queue;
    list_entry_t wait_link;
} wait_t;
```

他们各自相关的方法就不一一列了, 不然太长... 下面描述一下信号量的操作和semaphore_t里面变量的
意义

`value`变量用来记录可获取的资源数, > 0 时说明空闲, 当前信号量可以被获取, <= 0 时说明有线程正在
占用这个信号量, 不能被获取; `wait_queue` 就是个链表, 存储对应的 `wait` 型对象, 里面包含等待的
进程指针和相关的flag等等信息.

有关信号量的PV操作, P(wait) V(signal), 这个荷兰语的缩写实在是太别扭, 所以下面就说 等待 和 释放
把.

`void up(semaphore_t *sem)` 函数用来释放信号量, 内部使用 WT_KSEM flag调用 `__up` 函数实现, 这个
函数逻辑如下.

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```

大意就是关了中断, 如果等待队列空了, 就增加value值, 没空的话, 检查一下wait_state 然后唤醒对应
的进程, 顺手把这个wait从链表里摘除

`void down(semaphore_t *sem)` 用来获取信号量, 内部调用`__down`, 这个函数是阻塞等待的, 如果信
号量无法获取的话当前进程会被推入等待队列, 等待唤醒, 逻辑如下:

```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

大意是关了中断, 然后如果value > 0 (信号量可以被获取), 就回复中断状态直接返回, 不行的话, 把当前的
进程塞到等待队列里, 然后调用 `schdule()`. schedule之后其实就跳到了其他的线程, 所以对与这一段代码
来说, 从 schedule() 出来的时候其实已经成功的获取了 sem 信号量了, 所以把当前进程对应的wait从wait_queue
里摘下来, 然后检查一下wait_state, 直接返回就可以了.

`bool try_down(semaphore_t * sem)` 是非阻塞的尝试获取信号量, 如果sem可以被获取(value > 0), value -- 
然后返回 true, 否则返回false (为了防止被打断导致结果错误, 需要关中断), 因为不需要等待, 也就没有
将当前进程压入等待队列的需求, 逻辑就简单的多了, 代码如下:

```c
bool
try_down(semaphore_t *sem) {
    bool intr_flag, ret = 0;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --, ret = 1;
    }
    local_intr_restore(intr_flag);
    return ret;
}
```

**3. 请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。**

Linux里面比较常见的用户态信号量有两种: POSIX 信号量 和 System V 信号量. 我对POSIX信号量相对来说
比较熟悉, 所以就按照这个给出一个设计方案.

首先, 不管怎么说, 信号量实现肯定要涉及进程调度, 肯定需要内核态代码, 所以用户态相关的信号量操作需要
搞成系统调用的形式, 参考POSIX的设计, 给出API如下

    - sem_open: 打开或者创建一个信号量, 并且对它进行初始化
    - sem_close: 关闭并删除一个信号量
    - sem_getvalue: 拿到信号量的value值
    - sem_wait, sem_post: 信号量的PV操作函数

真正的POSIX信号量操作函数比上面列的多得多, 但是ucore也用不着那么多细致的区分, 所以想 sem_init, sem_destory,
sem_unlink... 之类的功能就都被我合并或者直接拿掉了.

为了能够让各个进程通过系统调用操作同一个信号量, 完成同步的目的, 需要在内核中专门划一段内存, 放所有的信号量的
数据.

综上, 内核态和用户态的信号量机制的不同主要有一下2点

    1. 内核态信号量可以直接调用相关的信号量操作函数, 并且直接完成进程切换; 用户态需要通过系统调用来完成
    2. 内核态的信号量直接放在内核栈上就可以了; 用户态信号量需要内核专门划一片共享内存来存

## 练习2


## 最后, 在读代码的时候的一点感想 -- 代码里的冗余传参和正确性检验

因为这次比较仔细的读了 wait.c 的实现, 发现了一点有意思的事. ucore里面非常喜欢传递用assert校验
来保障程序的正确性. 在wait_queue的实现里更是这样, 甚至在一些功能性函数里面传进去主逻辑用不到的
代码来进行额外的正确性校验, 下面举个栗子

```c
void
wait_queue_del(wait_queue_t *queue, wait_t *wait) {
    assert(!list_empty(&(wait->wait_link)) && wait->wait_queue == queue);
    list_del_init(&(wait->wait_link));
}
```

wait_queue_del这个函数功能相当简单, 就是把wait的link_list entry从queue里面摘除. 可以看出, 其实
完全没必要传递进 queue 参数, 只需要wait参数就足够了. queue就是用来校验一下, 这个wait 所在的queue
是不是调用者期望的那一个.

这么做客观上来讲其实是降低了ucore的性能的, 内存和cpu的占用都有了额外的开销, 即使后期禁用了 assert
这个负载也依然在(当然, 聪明的编译器能直接把它优化调, 就不会有额外开销啦). 但是我个人看到这个设计
是相当受启发的. 当然, ucore 其他地方的代码其实一直有这种技巧的使用, 只是之前没有注意到.

由于我自己开发的时候一般都是写高级语言, c++, python, js之类的, 像这种冗余校验保证正确性的操作没怎么
见过, 自己也从来不写, 但是我觉得这是一种好的风格, 最起码在某些情况下是一种保证逻辑正确性, 及时发现
漏洞的好的方法. 一方面, 这可以让程序员在写出错误逻辑的时候, 程序尽早的崩溃; 另一方面, 如果是程序员
设计本身有问题, ucore也可以立刻崩溃方便找到不足. 总之, 这种设计很好的体现了 "尽早崩溃" 的理念.

我自己写某些功能逻辑的时候也经常被一些脑抽的逻辑错误, 有时候干脆就是设计错误困住, 消耗大量时间. ucore
的这个设计给了我很大的提示, 以后我干活的时候也会考虑用上这种手段. 以前看编程技巧的文章的时候虽然也
看到过这种思想, 但是远没有 ucore 的这个实现给我来的印象深刻, 所以特意记录下来~




