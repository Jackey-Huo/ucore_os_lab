# LAB5

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

    1. 创建mm
    2. 为mm分配PDT，虚存的各种配合
    3. 复制各种代码，数据，bss等等段，elf格式解析，找入口
    4. 如果需要动态链接的话，链上，找入口
    5. 创建堆栈
    6. 各种设置寄存器以配置进程的特性
    7. 设置新进程的tf，找入口
    8. 然后就发生了一个中断，接下来就是硬件的锅啦


## 练习2

### 简要说明如何设计实现”Copy on Write 机制"

    1. 申请一个新的物理页
    2. 将目标页的内容复制上去
    3. 在当前进程的mm中，用新的页替换老的页

## 练习3

### 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的

进程的派生，执行，等待，结束

### 请给出ucore中一个用户态进程的执行状态生命周期图

TODO：疯了，一会儿写

