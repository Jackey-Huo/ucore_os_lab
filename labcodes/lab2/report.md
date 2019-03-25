# Lab2

## 练习1 first-fit 算法

原来给的代码中，default_alloc_pages, default_free_pages的插入删除位置都不对。通过gdb debug
把他们改对就没事了。尤其是default_free_pages里面，为了能够按顺序插入页，要多加你点逻辑.

（做完太久了，实在是想不起来了）


first-fit 的改进空间

    1. 感觉没什么改进空间。


## 练习2

### PDE 和 PTE 中每个组成部分的含义，及对ucore的潜在用处

这两个里存的都是 `physical address | flags`

flag主要3位，present，writable，user access。用来处理页错误，页的换入换出，非法读写等等


### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

跳到中断处理函数入口，按照终端号去找中断函数，将现场压到栈里

## 练习3

### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

有，Page的flags，读写关系

### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2

修改KERNBASE为0x0000000


