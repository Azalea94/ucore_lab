\# Lab1 report  
\# 2015-03-21 by dujuan  
\# 计24 2012011354 杜鹃

## [练习1]

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)


>1. 首先看Makefile文件中最终生成它的代码部分：

```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```
>2.为了生成ucore,先要生成kernel和bootblock，
>
>>2.1 生成kernel 的代码部分为：

```
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
>>>(看3.1)
>>2.2 生成bootblock的代码为：

```
bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	
```
3.1 为了生成kernel，依赖kernel.ld和KOBJS，第一个ld文件已经有了，所以需要KOBJS

```
KOBJS	= $(call read_packet,kernel libs)

```
>>这里的libs为文件夹中的libs里面的文件，这里涉及到的代码为：

```
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```
>>>这里的add_files_cc 的命令相当于

```
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
```
>>>而CFLAGS为：

```
CFLAGS	:= -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
```
>>>所以结果就是用上述配置的gcc命令产生一系列的*o文件

>>3.2 生成bootblock,需要bootfiles，也就是将boot文件夹下的bootmain和bootasm文件生成.o文件：

```
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```
>>还需要sign文件，生成sign的代码：

```
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```
>> 3.3 生成bootblock的执行代码是：

```  

	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	$(call create_target,bootblock)

```
>>>这几句话分别表示：1）输出信息 2）链接 3）将bootblock.obj编译成asm文件 4）将bootblock.obj文件复制成bootblock.out文件 5）用sign工具生成bootblock  

4.生成ucore的执行代码：

```

	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

```  

这几句的意思是：1）dd表示块复制 ，if=后面是源，of=后面是目标。count表示复制10000次 ，这句指令表示块的初始化； 2）将bootblock复制到ucore.img的第一个块中 ； 3）将kernel复制到ucore的从第二个块开始的部分，seek表示跳过n个块的大小。

[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

```
buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
    fclose(ofp);
    printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);
```
块大小为512字节，且最后两个字节是固定的 分别是0x55,0xAA

## [练习2]

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

单步跟踪，即调用gdb的stepi指令进行单步调式输出。主要步骤：  
1）运行qemu - S - s - hda . /bin/ucore. img - monitor stdio;  
2) 运行gdb并与qemu进行连接gdb ./bin/kernel
(gdb) target remote:1234  
3）进行stepi的单步调式  

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。  
在上一步的基础上进行gdb设置断点，再continue即可： 
 
```
b *0x7c00
c
```

[练习2.3] 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

在上一题的基础上加x /10i $pc
可输出反汇编得到的代码。

与原文件进行比较，看到是相同的


[练习2.4]自己找一个bootloader或内核中的代码位置， 设置断点并进行测试。  
设置断点memset:
```
b memset
c
```
可以用gdb -tui 进行代码的查看。

## [练习3]
分析bootloader 进入保护模式的过程。  
BIOS将硬盘第一个扇区的代码加载到0x7c00，开始执行，此时是实模式。即代码中的.code16

首先进行寄存器的初始化
```
	.code16
	    cli
	    cld
	    xorw %ax, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss
```

操作A20 Gate，使得地址访问空间大于1M：
```
	seta20.1:               # 等待8042Input buffer为空
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xd1, %al     # 发送write 8042输出端口的指令
	    outb %al, $0x64     #
	
	seta20.1:               # 等待8042Input buffer为空
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xdf, %al     # 将8042 Output Port（P2） 得到字节 的第2位置1， 然后写入8042 Input buffer；
	    outb %al, $0x60     # 
```

载入GDT表
```
	    lgdt gdtdesc
```

进入保护模式：通过将cr0寄存器相应位置变为1
```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

跳转到32位地址模式下

```
	 ljmp $PROT_MODE_CSEG, $protcseg
.code32
	protcseg:
```

在保护模式下，初始化寄存器和ebp、esp
```
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
```
调用bootmain函数，之后就是bootmain 加载操作系统了
```
	    call bootmain
```


## [练习4]
分析bootloader加载ELF格式的OS的过程。
  
在bootmain文件中  
readsect函数是从设备的第secno个扇区读取数据到dst,readseg是从设备读取任意长度的数据

在bootmain函数中，

```

	void bootmain(void) {
	    // 读取ELF的头部 并判断是否是ELF格式
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // 将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	   
	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```


## [练习5] 
实现函数调用堆栈跟踪函数 
代码见文件

输出中，堆栈最深一层为
```
	ebp:0x00007bf8 eip:0x00007d68 args:0x00007af0 0x00000001 0x00000007 0x00007d68 
    <unknow>: -- 0x00007d67 --

```

其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。栈底是0x7bf8(0x7c00-4)，所以压栈的时候ebp为栈底的位置，eip的值为call的下一条指令地址(bootasm.s的 jmp spin指令）；bootmain实际是没有参数的，所以args是从0x7c00开始之后的4个地址存的内容。下一行是执行call print_debuginfo(eip-1)，应该打印eip的上一条地址，因为是在.s文件中，所以显示unknown。



## [练习6]
完善中断初始化和处理

[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

IDT一个表项占用8字节，其中2-3字节是Selector，最低十六位和最高十六位组成offset，也就是gatedesc里的gd_off_15_0和gd_off_31_16.
由段选择子和位移得到入口地址。

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

见代码

[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

见代码


## [Challenge1]

增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），
当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务

根据第7节课视频对用户切换的介绍，可以得知，操作系统对不同状态的转换的一种方式是中断和中断恢复的方式。当中断恢复的时候通过弹出之前保留在栈中的寄存器信息来实现堆栈的转变和状态的切换。所以需要对堆栈中的标志位进行修改。  

在trap_dispatch中，将iret时会从堆栈弹出的段寄存器进行修改
	对TO User
```
	    tf->tf_cs = USER_CS;
	    tf->tf_ds = USER_DS;
	    tf->tf_es = USER_DS;
	    tf->tf_ss = USER_DS;
		tf->tf_eflags |= FL_IOPL_MASK;	
```
	对TO Kernel

```
	    tf->tf_cs = KERNEL_CS;
	    tf->tf_ds = KERNEL_DS;
	    tf->tf_es = KERNEL_DS;
		tf->tf_eflags &= ~FL_IOPL_MASK;
```

lab1_switch_to_user(void) ：
```
	asm volatile (
	    "sub $0x8, %%esp \n" //因为多保存了两个寄存器
	    "int %0 \n"
	    "movl %%ebp, %%esp" //修复esp
	    : 
	    : "i"(T_SWITCH_TOU)
	);
```

lab1_switch_to_kernel(void)：
```
	asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n" //修复esp
	    : 
	    : "i"(T_SWITCH_TOK)
	);
```
在idt_init中，将用户态调用SWITCH_TOK中断的权限打开。
	SETGATE(idt[T_SWITCH_TOK], 0, KERNEL_CS, __vectors[T_SWITCH_TOK], 3);





