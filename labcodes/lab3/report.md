# Lab3


---

## 练习1

### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。


这两个里存的都是 `physical address | flags`

flag主要3位，present，writable，user access。用来处理页错误，页的换入换出，非法读写等等

### 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

跳到中断处理函数入口，按照终端号去找中断函数，将现场压到栈里


---

# 练习2

## 现有的swap_manager框架是否足以支持在ucore中实现 extended clock

话说，这个是不是extended clock算法？

讲道理是可以的，但是需要改一下Page的数据结构，加入时钟位和改动位。其他的正常实现就可以
