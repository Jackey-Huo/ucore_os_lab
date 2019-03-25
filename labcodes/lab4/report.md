# LAB4

## 练习1

### 请说明proc_struct中struct context context和struct trapframe tf成员变量含义和在本实验中的作用是啥？

context 也叫上下文，其实就是寄存器值，每个线程切换的时候把当前寄存器的值压到内存里，下次再执行这个线程（进程）的时候再载入

trapframe 中断帧，进程切换其实是一个终端，返回这个进程的时候就相当与中断返回，有这个东西可以帮助回复现场

## 练习2

### 请说明ucore是否做到给每个新fork的线程一个唯一的id？

是的，唯一的。get_pid这个函数就是专门干这个事的。BUT，PID如果用完了好像就不是了

## 练习3

### 在本实验的执行过程中，创建且运行了几个内核线程？

2个，idleproc, initproc

### 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?

主要是暂时禁止系统切换进程的中断暂时，因为正在向proc切换，被打断就出错了
