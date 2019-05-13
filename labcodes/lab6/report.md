# LAB6 实验报告

## 一点代码分析

这几次实验比较重要的内容就是进程管理，今天看实验代码的时候仔细了些，注意到了proc_struct里面
标明进程之间相对位置的指针`cptr, yptr, optr`, 其实这3个指针在LAB5里就存在了, 但是当时做的不仔细
就没有仔细分析, 这次来分析一下

和这几个指针以及parent指针相关的最主要的函数有2个: `static void set_links(struct proc_struct *proc)` 和
`static void remote_links(struct proc_struct *proc)`

```c
// set_links - set the relation links of process
static void
set_links(struct proc_struct *proc) {
    list_add(&proc_list, &(proc->list_link));
    proc->yptr = NULL;
    if ((proc->optr = proc->parent->cptr) != NULL) {
        proc->optr->yptr = proc;
    }
    proc->parent->cptr = proc;
    nr_process ++;
}

// remove_links - clean the relation links of process
static void
remove_links(struct proc_struct *proc) {
    list_del(&(proc->list_link));
    if (proc->optr != NULL) {
        proc->optr->yptr = proc->yptr;
    }
    if (proc->yptr != NULL) {
        proc->yptr->optr = proc->optr;
    }
    else {
       proc->parent->cptr = proc->optr;
    }
    nr_process --;
}
```

从上面的逻辑就可以看出来, parent, cptr, optr, yptr这四个指针指向了当前进程相邻的4个进程, 空间位置可以
抽象成下面这样

```
                parent
                  ^
                  |
       yptr <-- proc --> optr
                  |
                  v
                cptr
```

其中, cptr是proc进程子进程的第一个, 其他的子进程都被挂在cptr所在的链上, 正常情况下cptr的左边(yptr的位置)
是空的, 子进程是从左往右排的, cptr指向左起第一个.

## 练习1

1. **请理解并分析sched_class中各个函数指针的用法，并结合Round Robin 调度算法描ucore的调度执行过程**

default_sched_class 的设置如下, 用法见每行代码后面的注释

```c
struct sched_class default_sched_class = {
    .name = "RR_scheduler",   # 不是函数指针, 一个名字
    .init = RR_init,          # 用来做调度算法的初始化, 各种list初始化之类的, 调用链 kern_init()->sched_init()->RR_init()
    .enqueue = RR_enqueue,    # 将进程压入等待队列
    .dequeue = RR_dequeue,    # 将进程弹出等待队列
    .pick_next = RR_pick_next,  # 找出下一个可以被执行的进程(不弹出)
    .proc_tick = RR_proc_tick,  # 每次发生时钟中断的时候调用, 递减time_slice变量, 归零了就将need_resched置1
};
```

Round-Robin算法其实就是轮询算法, ucore通过sched.c中的shcedule()函数将当前进程压入队列, 找到新的进程并执行, 如果没有新进程
执行, 则使用idleproc进程. RR算法主要用来管理进程的插入和下个进程的选取, 它非常简单, 调用enqueque则将进程向后插入, 调用next
就给出最前面的进程, 所以从外面来看, 就是一个FIFO.

2. **请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计**

多级反馈队列需要有多个容器来维护不同优先级的任务队列, 因此需要修改一下run_queue的定义, 假设我们的算法有3个优先级吧

```c
struct run_queue {
    list_entry_t run_list0;
    list_entry_t run_list1;
    list_entry_t run_list2;
    unsigned int proc_num;
    int max_time_slice0;
    int max_time_slice1;
    int max_time_slice2;
    // For LAB6 ONLY
    skew_heap_entry_t *lab6_run_pool;
}
```

为了记录process的优先级问题, 更改一下proc_struct的结构, 加一个记录优先级的标志位, 默认初始化为0

```c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
    int exit_code;                              // exit code (be sent to parent proc)
    uint32_t wait_state;                        // waiting state
    struct proc_struct *cptr, *yptr, *optr;     // relations between processes
    struct run_queue *rq;                       // running queue contains Process
    list_entry_t run_link;                      // the entry linked in run queue
    int time_slice;                             // time slice for occupying the CPU
    int proc_prority;                           // prority for schedule algorithm
    skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
    uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process 
    uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
};
```

入队, 出队和初始化其实和原来都差不多, 只不过入队的时候需要注意压到优先级合适的队里; 出队的时候注意调整优先级; 初始化
记得初始化全部list

```c
static void
RR_init(struct run_queue *rq) {
    list_init(&(rq->run_list0));
    list_init(&(rq->run_list1));
    list_init(&(rq->run_list2));
    rq->proc_num = 0;
}

static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_entry_t* suit_run_list = NULL;
    int suit_time_slice = 0;
    switch (proc->proc_prority) {
        case 0:
            suit_run_list = &(rq->run_list0);
            suit_time_slice = rq->max_time_slice0;
            break;
        case 1:
            suit_run_list = &(rq->run_list1);
            suit_time_slice = rq->max_time_slice1;
            break;
        case 2:
            suit_run_list = &(rq->run_list2);
            suit_time_slice = rq->max_time_slice2;
            break;
    }
    list_add_before(suit_run_list, &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > suit_max_time_slice) {
        proc->time_slice = suit_max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}

static void
RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
    if (proc->proc_prority < 2)
        (proc->proc_prority) ++;
}
```

处理时钟的proc_tick函数不会发生什么变化, 这里就不写了. 负责选取下一个运行进程的pick_next函数的
逻辑比较体现这个算法的思想, 代码如下

```c
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
    list_entry_t *le = NULL;
    if (!list_empty(&(proc->run_link0))) {
        le = list_next(&(rq->run_list0));
    } else if (!list_empty(&(proc->run_link1))) {
        le = list_next(&(rq->run_list1));
    } else if (!list_emptry(&(proc->run_link2))) {
        le = list_next(&(rq->run_list2));
    }

    if (le != NULL) {
      return le2proc(le, run_link);
    }

    return NULL;
}
```

上面代码的思路就是, 如果优先级高的队列里有等候线程, 那就把他们取出来执行, 优先级低的依次顺延.

其他的逻辑基本上不用怎么改, 多级反馈队列调度算法就可以了.

## 练习2

1. **简要协议下Stride-Scheduling算法的意思**

这个算法的中文解释确实比较少, 所以在这里多余解释一下它的意思. 这个算法其实就是为了让cpu给优先级高的
程序和新进入的程序多分时间片, 这样就能保证: 1. 耗时短的程序尽快执行完; 2. 同样耗时的情况下优先级高的
程序分到的时间片更多

为了实现这个目的, 设两个值: `stride`, `pass`, stride是个常数, 和优先级成反比, 每执行一次, pass就累加
一个stride. 每次挑进程的时候就挑那个pass值最小的执行. (**这里特别说一下, 实验手册上pass和stride的意义和
我这里写的正好反着, 但是我看这两个词的意义和paper里面的定义, 觉得应该是我写的这样, 但是实验代码还是按
ucore原来的实现的, 下面我回答问题用的pass的语义均是这一段我说的意思**)

2. 提问 1：如何证明`pass_max - pass_min <= BIG_STRIDE`

我们给所有进程的stride和pass编个号放进数组里: pass[], stride[]

在算法刚刚开始的时候, 所有pass[j] == stride[j], 那差小于BIG_STRIDE是自然的, 然后下面用数学归纳法来解释
一下(只是一个解释, 不是严格的形式化证明)

假设现在: 1. pass最大的进程编号为`k`; 2. pass最小的进程编号为`j0`, 第二小的为`j1`

new(pass[j0]) = pass[j0] + stride[j0]

现在分类讨论一下第二轮的情况

- pass最大的进程还是k, 最小的还是j0
	- 那这显然是满足的
- pass最大的进程是k, 最小的是j1
	- 也是满足的
- pass最大的进程变成j0了, 那最小的只能是j1

上面这种情况需要讨论一下

```
diff = pass[j0] + stride[j0] - pass[j1]
     = stride[j0] - (pass[j1] - pass[j0])
     <= BIG_STRIDE
```

这肯定也是小的, 证明完毕

3. **提问 2：在 ucore 中，目前Stride是采用无符号的32位整数表示。则BigStride应该取多少，才能保证比较的正确性？**

其实选择 `2^32 - 1`以内的数都成, 但是我觉得这个数好看, 那咱们就选 `0x7FFFFFFF`吧^_^

