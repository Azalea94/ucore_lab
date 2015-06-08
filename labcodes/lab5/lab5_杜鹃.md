# 操作系统Lab5实验报告

2012011354
计24
杜鹃

## 练习零：填写已有实验
使用understand的merge功能进行merge

## 练习 1: 加载应用程序并执行
do_execv函数调用load_icode（位于kern/process/proc.c中） 来加载并解析一个处于内存中的ELF执行文件格式的应用程序， 建立相应的用户内存空间 来放置应用程序的代码段、 数据段等， 且要设置好proc_struct结构中的成员 变量trapframe中的
内容， 确保在执行此进程后， 能够从应用程序设定的起始执行地址开始执行。 需设置正确的trapframe内容。

### 设计实现过程
根据代码中的注释进行赋值，使得在执行中断返回指令“iret”后， 能够让CPU转到用户态特权级， 并
回到用户态内存空间 ， 使用用户态的代码段、 数据段和堆栈， 且能够跳转到用户进程的第一条指令执行， 并确保在用户态能够响应中断。


### 和标准答案的差别
基本一致。

###请在实验报告中描述当创建一个用户态进程并加载了应用程序后， CPU是如何让这个应用程序最终在用户态执行起来的。 即这个用户态进程被ucore选择占用CPU执行（RUNNING态） 到具体执行应用程序第一条指令的整个经过。
答：内核调度进程，在执行中断返回指令“iret”(位于trapentry.S的最后一句)后，根据trapframe 能够让CPU转到用户态特权级， 并回到用户态内存空间 ，使用用户态的代码段、 数据段和堆栈， 执行中断返回指令“iret” 后， 将根据代码段位置切换到用户进程hello的第一条语句位置_start处.

## 练习二：父进程复制自己的内存空间给子进程
创建子进程的函数do_fork在执行中将拷贝 当前进程（即父进程） 的用户内存地址空间 中的合法内容到新进程中（子进程），完成内存资源的复制。 具体是通过copy_range函数（位于kern/mm/pmm.c中） 实现的， 请补充copy_range的实现， 确保能够正确执行。

### 设计实现过程
page 是被复制的页，npage是新开的页，是要新进程的空间。先通过page2kva函数获得两个page的kernel virtual address，然后调用memcpy函数将page的内容复制到npage。page_insert函数将物理页映射在了页表上  。

### 和标准答案的差别
命名区别。

###请在实验报告中简 要说明如何设计实现”Copy on Write 机制“， 给出概要设计， 鼓励给出详细设计。
见小组设计labx :)

## 练习三 阅读分析源代码， 理解进程执行 fork/exec/wait/exit 的实现， 以及系统调用的实现（不需要编码）

###请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？  
应用程序调用的exit/fork/wait/等库函数最终都会调用syscall函数， 只是调用的参数不同而已。  
1. 进程首先在 cpu 初始化或者 sys_fork 的时候被创建， 当为该进程分配了一个进程描 述符之后， 该进程进入 uninit态(在proc.c 中 alloc_proc)。当进程完全完成初始化之后， 该进程转为runnable态。  
2. exec时，加载程序，当到达调度点时， 由调度器 sched_class 根据运行队列rq的内容来判断一个进程是否应该被运行， 即把处于runnable态的进程转换成running状态， 从而占用CPU执行。  
3. running态的进程通过wait等系统调用被阻塞， 进入sleeping态。  
4. sleeping态的进程被wakeup变成runnable态的进程。  
5. running态的进程主动 exit 变成zombie态， 然后由其父进程完成对其资源的最后释放， 子进程的进程控制块成为unused。



###请给出ucore中一个用户态进程的执行状态生命周期图 （包执行状态， 执行状态之间 的变换关系， 以及产 生变换的事件或函数调用） 。 （字符方式画即可）
PROC_UNINIT     :            -- alloc_proc  
PROC_SLEEPING   :            -- try_free_pages, do_wait, do_sleep  
PROC_RUNNABLE   :            -- proc_init, wakeup_proc,   
PROC_ZOMBIE     :            -- do_exit
调用后面的函数后会变成前面的状态

## 重要知识点
用户进程的加载过程  
用户进程的内存管理：创建、 回收、 用户态和内核态的拷贝
进程的生命周期管理  
系统调用的流程  


##关系和差异：  

实验中会涉及到比较细致的实现问题。比如在创建进程中，会具体到mm/vma/stack/trapframe/地址映射关系 等方面。  
实验中貌似有不只一个虚拟地址？kernel virtual address 以及用户看到的逻辑地址。
