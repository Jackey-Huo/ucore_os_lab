# Lab1 实验报告

## 练习1

ucore.img是如何生成的

```
    1. 在Makefile中根据用户定义的各种宏设定编译器、模拟器、各种编译参数
    2. 设定各种目标文件的目录，相关的源文件
    4. 链接生成kernel文件，放在bin/kernel下面；它需要依赖bootasm.o, bootmain.o, sign，由宏批量产生

        bootfiles = $(call listf_cc,boot)
        $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

        $(kernel): $(KOBJS)
            @echo + ld $@
            $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
            @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
            @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
    5. 在sign工具的帮助下，链接生成bootblock文件，放在bin/bootblock下面
        $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
            @echo + ld $@
            $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
            @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
            @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
            @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
    6. 将kernel和bootblock按规则组织好，一起放到ucore.img文件里面
        $(UCOREIMG): $(kernel) $(bootblock)
            $(V)dd if=/dev/zero of=$@ count=10000
            $(V)dd if=$(bootblock) of=$@ conv=notrunc
            $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

硬盘主引导扇区的特征

```
    1. 引导区必须512字节，最后两个字节是 0x55 0xAA
```

## 练习2

1. 从CPU加电后的第一条指令开始，单步跟踪BIOS的执行

修改 gdbinit
```
set architecture i8086
target remote :1234
```

在bash下执行
```
make debug
```

2. 在初始化位置0x7c00设置实地址的断点
```
b *0x7c00
```

3. 将单步跟踪反汇编得到的代码与bootasm.S比较
gdbinit
```
break *0x7c00
continue
x /10i $pc
```

反汇编出来的代码
```
----------------
IN: 
0x00007c00:  fa                       cli      

----------------
IN: 
0x00007c01:  fc                       cld      

----------------
IN: 
0x00007c02:  31 c0                    xorw     %ax, %ax

----------------
IN: 
0x00007c04:  8e d8                    movw     %ax, %ds

----------------
IN: 
0x00007c06:  8e c0                    movw     %ax, %es

----------------
IN: 
0x00007c08:  8e d0                    movw     %ax, %ss

----------------
IN: 
0x00007c0a:  e4 64                    inb      $0x64, %al

----------------
IN: 
0x00007c0c:  a8 02                    testb    $2, %al

----------------
IN: 
0x00007c0e:  75 fa                    jne      0x7c0a

----------------
IN: 
0x00007c10:  b0 d1                    movb     $0xd1, %al

----------------
IN: 
0x00007c12:  e6 64                    outb     %al, $0x64

----------------
IN: 
0x00007c14:  e4 64                    inb      $0x64, %al
0x00007c16:  a8 02                    testb    $2, %al
0x00007c18:  75 fa                    jne      0x7c14

----------------
IN: 
0x00007c1a:  b0 df                    movb     $0xdf, %al
0x00007c1c:  e6 60                    outb     %al, $0x60
0x00007c1e:  0f 01 16 6c 7c           lgdtw    0x7c6c
0x00007c23:  0f 20 c0                 movl     %cr0, %eax
0x00007c26:  66 83 c8 01              orl      $1, %eax
0x00007c2a:  0f 22 c0                 movl     %eax, %cr0

----------------
IN: 
0x00007c2d:  ea 32 7c 08 00           ljmpw    $0x8:$0x7c32

----------------
IN: 
0x00007c32:  66 b8 10 00              movw     $0x10, %ax
0x00007c36:  8e d8                    movl     %eax, %ds

----------------
IN: 
0x00007c38:  8e c0                    movl     %eax, %es

----------------
IN: 
0x00007c3a:  8e e0                    movl     %eax, %fs
0x00007c3c:  8e e8                    movl     %eax, %gs
0x00007c3e:  8e d0                    movl     %eax, %ss

----------------
IN: 
0x00007c40:  bd 00 00 00 00           movl     $0, %ebp

----------------
IN: 
0x00007c45:  bc 00 7c 00 00           movl     $0x7c00, %esp
0x00007c4a:  e8 be 00 00 00           calll    0x7d0d

----------------
IN: 
0x00007d0d:  55                       pushl    %ebp
```

## 练习3

1. 为何及如何开启A20

为了兼容早期的PC，cpu的低20位默认是永远置0的，这会导致我们访问地址超过1MB的内存时出错，所以需要显式的关掉A20的内存时出错，所以需要显式的关掉A20。（说实话，我觉得用8042控制访存的地址线简直蠢的不行，但是鉴于这是上古程序员想出来的上古解决方案，而且我不行我也上不了，所以我就不多bb了。。。）


在bootasm.S中，通过向8042特定的port上写东西开启，汇编代码如下
```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

2. 如何初始化GDT表

设置好gdt表的数据排布，然后通过lgdt指令指向拍好的数据即可。bootasm.S中，gdtdesc有两段，代码如下
```
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

加载GDT的命令很简单
```
lgdt gdtdesc
```

3. 如何使能和进入保护模式

设好GDT之后，置CR0中的PE保护位为1即可，代码如下
```
movl %cr0, %eax
orl $CR0_PE_ON, %eax                                # CRO_PE_ON = 1
movl %eax, %cr0
```

## 练习4

1. bootloader如何读取硬盘扇区的

bootmain.c中，分别配置好扇区数，要读的地址，主从盘等参数，然后等待0x1F7(代码中使用的是通道1)，直到拿到想要的状态标志后通过insl指令从0x1F0地址向目标内存读数据。代码如下
```c
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

2. bootloader是如何加载ELF格式的OS

在libs/elf.h头文件中，定义了struct elfhdr，用来描述elf文件格式。

cpu将磁盘中的elf文件读取到内存中后，按照elfhdr数据结构的格式来解析这块内存，并且做相应的操作（对比ELF_MAGIC，通过函数调用的方式跳到entry point函数中等等）。

## 练习5

补全print_stackframe之后得到的输出如下：
```
...
ebp:0x00007b38 eip:0x00100a3c args:0x00010094 0x00010094 0x00007b68 0x0010007f
    kern/debug/kdebug.c:305: print_stackframe+21
ebp:0x00007b48 eip:0x00100d30 args:0x00000000 0x00000000 0x00000000 0x00007bb8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b68 eip:0x0010007f args:0x00000000 0x00007b90 0xffff0000 0x00007b94
    kern/init/init.c:48: grade_backtrace2+19
ebp:0x00007b88 eip:0x001000a1 args:0x00000000 0xffff0000 0x00007bb4 0x00000029
    kern/init/init.c:53: grade_backtrace1+27
ebp:0x00007ba8 eip:0x001000be args:0x00000000 0x00100000 0xffff0000 0x00100043
    kern/init/init.c:58: grade_backtrace0+19
ebp:0x00007bc8 eip:0x001000df args:0x00000000 0x00000000 0x00000000 0x00103280
    kern/init/init.c:63: grade_backtrace+26
ebp:0x00007be8 eip:0x00100050 args:0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+79
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d6d --
...
```

对于每一个函数栈，ss:ebp存储调用函数的ebp位置，ss:ebp+4存储返回地址，即调用函数的eip位置

输出的前几条记录都是kernel中的函数，最后面位于0x0007bf8的函数栈应该是属于bootloader的bootmmain函数的，因为它的调用位置在0x00007c00,所以call的时候压栈到这个位置。



## 练习6

1.中断描述符表中一个表项多少字节？哪几位代表中断处理代码的入口

8字节;0-1和6-7为位移, 2-3为段选择子，这两项决定了中断处理代码的入口



代码补全之后，时钟和键盘中断的输出如下

```
100 ticks
kbd [000]
kbd [100] d
kbd [000]
kbd [100] d
kbd [000]
100 ticks
```
