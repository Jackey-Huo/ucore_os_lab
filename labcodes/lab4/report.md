# Lab4 实验报告

## 练习1

设计实现过程：

把能初始化的全初始化了，基本上什么有用的都不知道，所以都不填。。。

特别注意cr3，这个寄存器和虚地址映射有关，需要填在pmm.c里面定义的boot_cr3

```
  proc->state = PROC_UNINIT;
  proc->pid = -1;
  proc->runs = 0;
  proc->kstack = 0;
  proc->need_resched = 0;
  proc->parent = NULL;
  proc->mm = NULL;
  memset(&(proc->context), 0, sizeof(struct context));
  proc->tf = NULL;
  proc->cr3 = boot_cr3;
  proc->flags = 0;
  memset(proc->name, 0, PROC_NAME_LEN);
```

1. proc_struct中 `struct context context` 和 `struct trapframe *tf` 成员变量含义和在实验中的作用

context 用于进程切换时各种寄存器数据的存取，就是我们常说的上下文

trapframe 用于保存调用新进程前的内核信息，之后可以用来参与进程结束后的返回


## 练习2

设计实现过程：

先调用`alloc_proc`给进程分配空间，调用`setup_kstack`分配内核栈的内存，复制内存管理`copy_mm`,
`copy_thread`复制trapframe

之后调用`local_intr_save`宏关闭中断，给进程分配pid，存储到proc_list和hash_list里面，然后调用`local_intr_restore`
恢复中断寄存器

调用`wakeup_proc`来唤醒新进程，返回pid号

1. 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

能的，get_pid函数会不断找一个PID号的区间，以供新的进程使用。又因为PID号的个数比MAX_PROCESS多，所以是可以的.

```c
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```


## 练习3

1. 本实验的执行过程中，创建且运行了几个内核线程

2个，一个idleproc，一个是通过kernel_thread 创建的进程

2. 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由

用来关闭中断（通过写中断寄存器）和回复之前的中断寄存器

```
static inline bool
__intr_save(void) {
    if (read_eflags() & FL_IF) {
        intr_disable();
        return 1;
    }
    return 0;
}

static inline void
__intr_restore(bool flag) {
    if (flag) {
        intr_enable();
    }
}

#define local_intr_save(x)      do { x = __intr_save(); } while (0)
#define local_intr_restore(x)   __intr_restore(x);
```

通过读写EFLAGS的对应位来开关中断

## 我的答案与参考答案的区别

1. 一开始没有开关中断，看了答案之后改正了

2. `copy_mm`函数的返回至是写死的，所以一开始我并没有检查它的输出，看了答案之后觉得答案写的甚好，我就改了

## OS原理中很重要但是没有对应的点

进程切换，实验代码里有，但是联系里没有涉及
