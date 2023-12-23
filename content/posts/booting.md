+++
title = 'OSV启动流程分析-boot16.S'
date = 2023-12-22T22:31:54+08:00
draft = false 
tags =["osv","汇编语言","bootloader"]
categories=["osv"]
+++
{{< toc >}}

# OSV的启动流程

常见的操作系统启动流程

ROM ->LOADER ->RUNTIME --->BOOTLOADER ->OS  

机器上电后，所需要的第一个步骤就是从ROM中加载BIOS，然后进行硬件自检(Power On Self Test)，检查计算机需要的最基本的硬件(CPU、内存、磁盘、键盘等)。然后根据BIOS中的相关启动顺序的配置，进行加载LOADER。默认情况下，是查找磁盘的第一个扇区(Master Boot Record,MBR)。

硬盘上第0磁道第一个扇区被称为 MBR，也就是 Master Boot Record，即主引导记录，它的大小是512字节，可里面却存放了预启动信息、分区表信息。可分为两部分：

第一部分为引导（PRE-BOOT）区，占了446个字节；

第二部分为分区表（PARTITION TABLE），共有64个字节，记录硬盘的分区信息。

预引导区的作用之一是找到标记为活动（ACTIVE）的分区，并将活动分区的引导区读入内存。剩余两个字节(0x55,0xaa)为结束标记。

## osv-boot

在osv中 boot16.S即为一般计算机上的MBR，

首先其入口点为start,跳转到init

```gas
start:
    ljmp $0, $init
```

```gas 
init:
    rdtsc   ## 读取时间戳，为一个64位的数字，高32位存在edx中，低32位存在eax中
    mov %eax, mb_tsc1_lo  ##保存时间戳低位
    mov %edx, mb_tsc1_hi   ## 保存时间戳高位
    xor %ax, %ax      ##将ax 置0 
    mov %ax, %ds        ## ds es ss寄存器置0 
    mov %ax, %es
    mov %ax, %ss
    mov $0x7c00, %sp          ## 初始化栈寄存器为0x7c00 
    mov $0x2401, %ax # enable a20 gate
    int $0x15                ## 向BIOS中断0x15传递0x2401参数，启用A20门，它会影响内存地址线的第20位，允许访问1MB以上的内存
    lea int1342_boot_struct, %si                ##将int1342_boot_struct的地址加载到si索引寄存器中
    mov $0x42, %ah                       ## ah=0x42表示从硬盘中读取数据
    mov $0x80, %dl                       ## dl中表示硬盘drive index,0x80表示第一个硬盘
    int $0x13
    movl $cmdline, mb_cmdline
```
继续从硬盘中读取数据

```gas
read_disk:
    lea int1342_struct, %si     
    mov $0x42, %ah
    mov $0x80, %dl
    int $0x13
    jc read_error # Notify the kernel about read errors

    cli         ##关中断
    lgdtw gdt    ##   加载全局描述符 GDT
    mov $0x11, %ax
    lmsw %ax        ## load Machine  Status Word ，写入CR0寄存器低16位
    ##0x11即00010001,第0位置1表示开启保护模式，第4位置1表示使用的是387浮点协处理器否则是287协处理器。
    ##On the 386, it allowed to specify whether the external math coprocessor was an 80287 or 80387 
    ljmp $8, $1f    ##长跳转，$1f表示当前位置下面的"1"标签的地址。
    ##虽然设置了CR0，但当前执行的代码仍处于16位实模式下。该条指令的作用是切换段地址，进入保护模式
1:
    .code32     ##进入32位保护模式
    mov $0x10, %ax ##将选择子0x10移动到寄存器ax中
    mov %eax, %ds   ##使得数据段寄存器 ds 指向选择子 0x10 对应的数据段
    mov %eax, %es   ##使得额外段寄存器es 指向0x10对应的数据段
    mov $tmp, %esi   ##将tmp（0x8000）加载到源变址寄存器中
    mov xfer, %edi  ##将变量 xfer 的地址（偏移）加载到目的变址寄存器 edi
    mov $0x8000, %ecx   ##将常数0x8000移动到计数寄存器ecx中。这个值将用于指定数据传输的字节数。
    rep movsb    ##重复执行 movsb 指令，将 ecx 指定数量的字节从 esi 复制到 edi
    mov %edi, xfer   ##更新xfer索引
    mov $0x20, %al   
    mov %eax, %ds
    mov %eax, %es
    ljmpw $0x18, $1f
1:
    .code16
    mov $0x10, %eax
    mov %eax, %cr0   ##重新进入实模式
    ljmpw $0, $1f
1:
    xor %ax, %ax
    mov %ax, %ds
    mov %ax, %es   ##置0 
    sti     ##开中断  
    addl $(0x8000 / 0x200), lba ## lba增加512字节，指向下一个扇区
    decw count32   ##计数器减1 
    jnz read_disk   ##不为0则跳转到read_disk位置继续读取下一个扇区
    jmp done_disk   ##为0则完成读取，跳转到done_disk 
read_error:
    mov %ah, mb_disk_err
done_disk:
    rdtsc
    mov %eax, mb_tsc_disk_lo
    mov %edx, mb_tsc_disk_hi   ##记录读取完磁盘的时间戳

    mov $e820data, %edi   ##将e820data的地址传入edi寄存器
    mov %edi, mb_mmap_addr     ## 将e820data的地址写入 mb_mmap_addr中 
    xor %ebx, %ebx

```

```gas 
more_e820:
    ##设置 int 0x15   ax=0xe820 的传入参数
    mov $100, %ecx                  ##指定ARDS结构的字节大小
    mov $0x534d4150, %edx           ##固定签名标记,这个16进制是SMAP的ASCII码
    mov $0xe820, %ax              
    add $4, %edi    ##ES:DI是ARDS的缓冲区，BIOS会将获取到的内存信息写入EDI指向的地址
    int $0x15
    jc done_e820       ##CF标志为1表示读取失败，跳转到done_e820 
    mov %ecx, -4(%edi)    ##ecx寄存器中保存着写入的字节长度  
    add %ecx, %edi        ##edi保存的地址应当增加相对应的大小
    test %ebx, %ebx    ##  ebx保存着后续内存段信息的数量，ebx为0表示这是最后一个ARDS 
    jnz more_e820
done_e820:
    sub $e820data, %edi     ##完成写入，将edi和最初的e820data地址相减，得到写入的长度
    mov %edi, mb_mmap_len

    cli   ##关中断，并切换到保护模式
    mov $0x11, %ax
    lmsw %ax
    ljmp $8, $1f   
1:
    .code32
    mov $0x10, %ax
    mov %eax, %ds
    mov %eax, %es
    mov %eax, %gs
    mov %eax, %fs
    mov %eax, %ss
    call *lzentry
    rdtsc
    mov %eax, mb_uncompress_lo
    mov %edx, mb_uncompress_hi
    mov $mb_info, %ebx
    call *start32    ##跳转到boot.S中的start32标签

```

总结boot16.S的工作:
- 开启A20总线，以支持更大的地址空间
- 从磁盘中加载osv的loader
- 利用int 0x15 AX=0xe820 向BIOS获取内存大小信息，并存入e820data中
- 跳转到boot.S  


### A20总线
我们知道，8086/8088系列的CPU，在实模式下，按照段地址:偏移地址的方式来寻址。这种方式可以访问的最大内存地址为0xFFFF:0xFFFF，转换为物理地址0x10FFEF。而这个物理地址是21bit的，所以为了表示出这个最大的物理地址，至少需要21根地址线才能表示。

然而，8086/8088地址总线只有20根。所以在8086/8088系列的CPU上，比如如果需要寻址0x10FFEF，则会因为地址线数目不够，被截断成0x0FFEF。再举个例子，如果要访问物理地址0x100000，则会被截断成0x00000。第21位会被省略。也就是说地址不断增长，直到0x100000的时候，会回到“0x00000”的实际物理地址。这个现象被称为“回环”现象。这种地址越界而产生回环的行为被认为是合法的，以至于当时很多程序利用到了这个特性（比如假定访问0x100000就是访问0x00000）。

A20总线用来控制第21位（如果最低位编号为0，那第21位的编号就是20）及更高位是否有效。实际上可以想象成，第21位（及更高位）都接入了一个和A20总线的与门。当A20总线为1，则高位保持原来的。当A20总线为0，则高位就始终为0。这样，当A20总线为0的时候，8086/8088的回环现象将会保持。这么一来旧程序就可以兼容了。



# Reference 

[X86 汇编语言 - 内存检测](http://blog.ccyg.studio/article/809f4839-9ad3-4a66-a599-aa445b915157/) 
