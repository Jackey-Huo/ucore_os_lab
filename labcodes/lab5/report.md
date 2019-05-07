# LAB5

## 练习0

按照惯例,对LAB1/2/3/4的代码进行补全即可.

需要注意的是,由于涉及到调度问题,在trap.c文件中每个TICK_NUM个时钟之后,需要将current的need_resched参数置1

另外,在init_idt函数中,初始化了全部中断向量后,需要把T_SYSCALL的特权级置为DPL_USER,使得用户态程序能够发起系统调用

## 练习1

### 设计实现过程

```c
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
```

### 用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过

大部分进程都是通过schedule()函数调用proc_run()跑起来的.在proc_run()里面,先加载目标进程的esp和cr3寄存器,
再恢复目标进程的上下文(各种寄存器的值),并保存正在执行的进程的上下文,然后cpu就可以正常的跳到目标进程里去了.

代码如下,特别的,为了防止切换进程的操作被打断,代码中还特意上了锁.

```c
    struct proc_struct *prev = current, *next = proc;
    local_intr_save(intr_flag);
    {
        current = proc;
        load_esp0(next->kstack + KSTACKSIZE);
        lcr3(next->cr3);
        switch_to(&(prev->context), &(next->context));
    }
    local_intr_restore(intr_flag);
```


## 练习2

### 简要说明如何设计实现”Copy on Write 机制"

在mm外面再包一层结构,让它有存储额外的页的功能(比如mm_extern).每个进程持有mm_extern.
当某个进程对内存有写操作的时候,申请一个新的页,将原始内存的内容复制到新页中,然后对新
的页进行写操作.

当需要写回时,将mm_extern中单独维护的页写回到原始页中

## 练习3

### 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的

#### fork

实验中使用do_fork函数创建一个新的进程,并且调用wakeup_proc唤醒这个新的进程,但是cpu并不
是直接跳到新进程里的.

do_fork里还承担了一些proc_struct的配置工作,比如将父进程的mm控制块复制到新进程中,为新
进程分配栈空间,为新进程申请pid等等

#### exec

实验中do_execve实现exec功能,这个函数主要负责加载用户程序(ELF文件),通过syscall调用

#### wait

实验中wait的功能由do_wait函数实现.这个函数和linux中的waitpid功能很像(pid != 0时,
pid == 0时像linux里的wait),逻辑是等待某个 特定pid且是当前进程子进程的进程,如果
子进程状态是ZOMBIE,将它从进程哈西表和list表中删除, 释放有关的栈空间；如果子进
程还没结束,阻塞着等它知道子进程结束.

在实验中,do_wait函数也是通过syscall调用的

#### exit

实验中exit的功能由do_exit实现,这个函数先将mm的引用数减一,如果mm没有引用了,就调用相关
函数释放掉mm相关的东西.然后将当前进程的状态置为ZOMBIE态.之后遍历当前进程的所有子进程,
将它们挂在initproc进程下(如果是ZOMBIE状态的子进程,直接通过initproc杀掉)

在实验中,除了可以通过syscall调用do_exit函数之外,其他进程控制相关的函数当执行遇到错误
时也会直接调用它销毁当前进程

### 请给出ucore中一个用户态进程的执行状态生命周期图

```
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  +
                                           -----------------------wakeup_proc----------------------------------

```


