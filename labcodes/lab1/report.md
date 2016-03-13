# uCore OS Lab1 Report
@author 张敬卿 2012011380

## 练习一
### 1. Makefile解读

> 1. 检查GCCPREFIX和QEMU是否可以正常使用

> 2. 定义编译器和参数
>> 1. HOSTCC和CC定义了使用```gcc```或者```clang```编译器
>> 2. HOSTCFLAGS的参数：```-g```生成调试信息 ```-Wall```显示警告信息 ```-O2```进行o2优化
>> 3. CFLAGS的参数：```-fno-builtin```不对内建的C函数进行优化和处理 ```-Wall```显示警告信息 ```-ggdb```生成gdb调试信息 ```-m32```适用于32位环境 ```-gstabs```生成stabs调试信息 ```-nostdinc```不使用标准库 ```-fno-stack-protector```不针对堆栈进行保护
>> 4. LDFLAGS的参数：```-m```模拟为i386上的连接器 ```-nostdlib```不使用标准库

> 3. 生成kernel
>> 1. 首先```$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))```生成包括init.o stdio.o string.o等文件，编译的参数如上CFLAGS所示，通过通过-I链接kern和libs里面的代码文件
>> 2. 然后```$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)```生成kernel文件，通过```-T```运行制定kernel.ld文件，链接如上编译的.o文件

> 4. 生成bootblock 
>> 1. 需要首先编译boot文件夹下的bootmain.c和bootasm.S，分别生成bootmain.o和bootasm.o，直接用```$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(HOSTCC),$(CFLAGS) -Os -nostdinc))```批量生成，参数使用同CFLAGS，新增-Os参数针对代码大小进行优化，保证bootloader不能大于510Bytes（最后两个字节是指定的标准结尾）
>> 2. 然后需要编译sign.c，先生成sign.o文件```$(call add_files_host,tools/sign.c,sign,sign)```
在生成sign二进制文件```$(call create_target_host,sign,sign)```，两条指令都只是用了```-g -Wall -O2```三个参数
>> 3. 用```$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)```编译bootblock，使用LDFLAGS的参数，另外有```-N```设置代码段和数据段均可读写 ```-Ttext```制定代码段开始位置 ```-e```指定入口
>> 4. 用```@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)```拷贝bootblock.o到bootblock.out，其中参数```-S```移除所有符号和重定位信息 ```-O```指定输出格式
>> 5. 最后```@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)```通过sign工具和bootblock.out生成bootblock二进制文件

> 5. 生成ucore.img的镜像
>> 1. 生成镜像的前提是kernel和booblock都编译成功
>> 2. ```dd if=/dev/zero of=bin/ucore.img count=10000```创建10000个块，每个块512Bytes，默认填入零
>> 3. ```dd if=$(bootblock) of=bin/ucore.img conv=notrunc```第一个块填入bootblock
>> 4. ```dd if=$(kernel) of=bin/ucore.img seek=1 conv=notrunc```之后填入kernel

### 2. 符合规范的硬盘主引导扇区的特征
> 1. 引导扇区是512Bytes，引导扇区的结尾有特殊的要求，第510个字节（倒数第二个字节）是0x55，第511个字节（最后一个字节）是0xAA
> 2. 对于bootblock如上编译的规范中提到，代码段是从0x7C00开始的


## 练习二
### 1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行
> 1. 在tools/gdbinit里添加```target remote :1234```
> 2. 运行gdb```$ make debug```
> 3. 单步调试```stepi```
> 4. 查看当前指令和下一条指令```x /2i $pc```

### 2. 在初始化位置0x7c00设置实地址断点,测试断点正常
> 1. 在tools/gdbinit里面添加```b *0x7c00```
> 2. ```$ make debug```启动并```continue```后看到0x7c00代码段开始
```
0x7c00  cli
0x7c01  cld
0x7c02  xor     %eax, %eax
0x7c04  mov     %eax, %ds
0x7c06  mov     %eax, %es
```

### 3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较
> 单步调试的代码与bootasm.S的代码相同

### 4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试
> 在tools/gdbinit里面添加```b kern_init```可以测试内核初始化的断点


## 练习三
### 1. 为何开启A20，以及如何开启A20
> A20是IBM为了兼容8086/8088（20根地址线=1M）和80286（24根地址线=16M），如果A20关闭，如同8086/8088一样，机器只有1M的内存空间可以使用；如果A20开启，才可能利用所有的内存空间（32位机器就是4G空间）；因此进入保护模式之前需要开启A20
> bootasm.S开启A20的方式是通过8042寄存器也就是通过键盘控制器开启A20
```
seta20.1:
    inb $0x64, %al      // 等待8042寄存器闲置
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al     // 发出写数据到8042寄存器P2端口的指令
    outb %al, $0x64 

seta20.2:
    inb $0x64, %al      // 等待8042寄存器闲置
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al     // 写入0xdf，设置A20为1，打开A20
    outb %al, $0x60     
```

### 2. 如何初始化GDT表

```
// 读入GDT表格
lgdt gdtdesc
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0

// 长跳转到下一条指令，但是已经进入到32位的模式
ljmp $PROT_MODE_CSEG, $protcseg
```

### 3. 如何使能和进入保护模式

```
// 设置段寄存器
movw $PROT_MODE_DSEG, %ax
movw %ax, %ds
movw %ax, %es
movw %ax, %fs
movw %ax, %gs
movw %ax, %ss
// 初始化堆栈
movl $0x0, %ebp
movl $start, %esp

// 进入保护模式，可以调用boot了
call bootmain
```

## 练习四
### 1. bootloader如何读取硬盘扇区的？
```C
/* 从硬盘里读一个扇区 */
static void
readsect(void *dst, uint32_t secno) {
    waitdisk();

    outb(0x1F2, 1);                         // 设置成一个扇区
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // 读扇区

    waitdisk();

    // 读到的制定的位置
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

### 2. bootloader是如何加载ELF格式的OS？
```C
/* 加载OS，bootloader的入口 */
void
bootmain(void) {
    // 读取ELF头部
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    //判断是否合法
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // 一段一段读取OS程序
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 调用ELF头部规定的OS入口，没有返回的
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);
    while (1);
}
```


## 练习五
> 实现详见kern/debug/kdebug.c::print_stackframe
### 最后一行含义
```
bootloader代码段定义从0x7c00开始，堆栈从0x7c00往下开始，在call bootmain的时候有压栈，占据8个字节（一个返回地址四字节，一个初始的ebp=0四字节），因此是最后一行的ebp=0x7bf8也即是bootmain的ebp
```

## 练习六
### 1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
> 一个表项8个字节，第3-4字节的段选择子查表获得基地址，第1-2字节和第7-8字节获得偏移，一起获得代码入口

### 2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。
> 实现请见代码

### 3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。
> 实现请见代码




