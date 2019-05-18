# LAB7 实验报告

## 练习1

**对比一下练习0之后的代码和lab6有什么不同.**

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


**请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程。**



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




