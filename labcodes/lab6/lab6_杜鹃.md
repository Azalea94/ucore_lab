# 操作系统Lab6实验报告

2012011354
计24
杜鹃

## 练习零：填写已有实验
使用understand的merge功能进行merge  
需要修改之前lab的地方有：  
1. proc.c 中alloc_proc 函数里在对proc进行初始化时，加上lab6的部分。其中包括list实现的部分和斜堆实现的部分。  
2. trap.c中 加上对sched_class_proc_tick函数的调用  
```

   assert(current != NULL);
   sched_class_proc_tick(current);
   //current->need_resched = 1;


```
3. sched_class_proc_tick 函数的static去掉，并在sched.h中声明。

## 练习 1: 使用 Round Robin 调度算法
make grade 之后，大部分样例都是对的，其中priority和waitkill出现wrong错误，其中priority错误正常。waitkill如果单独make run-waitkill，原本出现错误的地方是可以正确的。  

### 请理解并分析sched_calss中各个函数指针的用法， 并接合Round Robin 调度算法描ucore的调度执行过程  
各个函数指针的实现：  
1.RR_enqueue：把某进程的进程控制块指针放入到rq队列末尾， 且如果进程控制块的时间 片为0， 则
需要把它重置为rq成员 变量max_time_slice。 这表示如果进程在当前的执行时间 片已经用完， 需要等到下一次有机会运行时， 才能再执行一段时间 。  
2. RR_pick_next:选取就绪进程队列rq中的队头队列元素， 并把队列元素转换成进程控制块指针。  
3. RR_dequeue：把就绪进程队列rq的进程控制块指针的队列元素删除， 并把表示就绪进程个数的
proc_num减一。  
4. RR_proc_tick：每次timer到时后， trap函数将会间接调用此函数来把当前执行进程的时间片
time_slice减一。 如果time_slice降到零， 则设置此进程成员 变量need_resched标识为1， 这样在下一次中断来后执行trap函数时， 会由于当前进程程成员 变量need_resched标识为1而执行schedule函数， 从而把当前执行进程放回就绪队列末尾， 而从就绪队列头取出在就绪队列上等待时间 最久的那个就绪进程执行。  
执行过程：  
RR算法在default_sched.h,default_sched.c实现。RR调度算法的调度思想是让所有runnable态的进程分时轮流使用CPU时间 。 RR调度器维护当前runnable进程的有序运行队列。 当前进程的时间片用完之后， 调度器将当前进程放置到运行队列的尾部， 再从其头部取出进程进行调度。 RR调度算法的
就绪队列在组织结构上也是一个双向链表, 在进程控制块proc_struct中增加了一个成员变量time_slice， 用来记录进程当前的可运行时间 片段。在每个timer到时的时候， 操作系统会递减当前执行进程的time_slice， 当time_slice为0时， 就意味着这个进程运行了一段时间 （这个时间 片段称为进程的时间 片） ， 需要把CPU让给其他进程执行， 于是操作系统就需要让此进程重新回到rq的队列尾， 且重置此进程的时间 片为就绪队列的成员 变量最大时间 片max_time_slice值， 然后再从rq的队列头取出一个新的进程执行。   
 RR的特点是 round-robin 调度器， 在假设所有进程都充分使用了其拥有的 CPU 时间 资源的情况下， 所有进程得到的 CPU 时间 应该是相等的。 

###请在实验报告中简 要说明如何设计实现”多级反馈队列调度算法“， 给出概要设计， 鼓励给出详细设计  
设计：维护多个（比如4个）就绪队列，这些就绪队列的特点是：1）优先级不同，设定为P（Q1)>P(Q2)>P(Q3)>...  2)每个队列的运行时间片以一定比例递增（比如2倍）。这样保证各个队列的时间片是随着优先级的增加而减少的，也就是说，优先级越高的队列中它的时间片就越短。  3）一个队列中的时间片是相同的。  
算法：  
1）进入队列时，先进入优先级最高的Q1；  
2）调度时，先调度优先级高的队列，也就是说Q1为空时调度Q2，以此类推。  
3）一个作业如果在Q1的一次执行中，时间片用完了还没有做完，则进入下一个优先级队列，一直到最后一级做完。  


### 和标准答案的差别
trap.c中trap_dispatch函数中对sched_class_proc_tick函数的调用。    
proc.c中alloc_proc函数中，答案是`proc->run_link.prev=proc->run_link.next=NULL`；我的是 `proc->run_link.prev = proc->run_link; proc->run_link.next = proc->run_link;` 因为在enqueue函数中有一个`assert(list_empty(&(proc->run_link)));` 判断是不是自环，如果不是会报错，所以这里答案有错。     



## 练习二：练习 2: 实现 Stride Scheduling 调度算法（需要编码）
首先需要换掉RR调度器的实现， 即用default_sched_stride_c覆盖default_sched.c。 然后根据此文件和后续文档对Stride度器的相关描述， 完成Stride调度算法的实现。


### 设计实现过程
将default_sched_stride_c改为default_sched_stride.c , 新建一个对应的.h头文件。将.c中的sched_class 由default_sched_class 改为 stride_sched_class ，修改对应的.h文件中的声明，以及sched.c中的调用。   

函数指针的实现：  
stride_enqueue 和stride_dequeue比较简单，与RR基本相同。  
主要实现的stride算法的函数是stride_pick_next，它主要做两件事：1）扫描整个运行队列， 返回其中stride值最小的对应进程。
2）更新对应进程的stride值， 即pass = BIG_STRIDE / P->priority; P->stride += pass。

proc tick函数:检测当前进程是否已用完分配的时间 片。 如果时间片用完， 设置进程结构的相关标记来引起进程切换。   

另外，需要设置BIG_STRIDE为0x7fffff(有符号数中的最大值） 

PS : 为了使得priority输出结果稳定，将时间片由1000增大到10000.

## 重要知识点
操作系统的调度管理机制  
进程的生命周期  
整个操作系统对sched的调用情况
时间片的实现（timer）  
stride算法中溢出的处理和原理


##关系和差异：  

实验中一个比较重要的思想是：框架和实现。框架实际上是对实现的一种抽象：  
将进程变化的情况就抽象为
调度器相关的一个变化感知操作： timer时间事件感知操作。   
这样在进程运行或等待的过程中， 调度器可以调整进程控制块中与进程调度相关的属性值（比如消耗的时间 片、 进程优先级等） ， 并可能导致对进程组织形式的调整（比如以时间 片大小的顺序来重排双向链表等） ， 并最终可能导致调选择新的进程占用CPU运行。 