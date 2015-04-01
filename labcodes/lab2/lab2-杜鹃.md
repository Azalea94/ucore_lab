#lab2 实验报告  
计24 2012011354 杜鹃

##[练习一]  
###实现 first-fit 连续物理内存分配算法  

答：设计思路：  
1）首先了解涉及到的类和功能实现方式：  
 > Page 页；双向链表；   
 > 管理的方式是通过设定page的状态位，free,reserved，ref等  
 > 需要实现的函数：init_memmap/alloc_pages/free_pages

2) init_memmap实现：  
>函数功能是根据现有的内存情况构建空闲块列表的初始状态；传入参数：某个连续地址的空闲块的起始页，页个数；  
>实现：从起始页开始进行连续n个页的初始化，初始化的工作包括：设置flags property ref等，并将其添加到空闲链表中。  
>最后将起始页的property设置为n大小，nr_free大小增加n大小。

3）alloc_pages实现：  
>first-fit算法，找到第一个符合大小的块，重新计算空闲块大小，返回分配的块的首地址  
>实现：从空闲列表头开始进行检查，通过property属相检查是否大小符合要求  
>如果找到则进行状态位的改变，将找到的位置到n大小的空间移出空闲列表。重新计算剩余的空闲块大小、nr_free大小。  

4）free_pages实现：
>相当于alloc函数的逆过程  
>首先根据地址大小找到回收块应该插入的位置（因为空闲块是按照地址大小排列的），插入空闲列表。  
>将状态位进行改变，nr_free增加n大小  
>进行前后的检查是否进行merge操作

###可改进的地方：
在free函数中进行回收块地址的查询过程可进行优化，现在的实现是线性查找，可以改进使用二分查找，因为空闲块是按照地址大小排列的。  
###与参考答案比较：  
基本相同，我在alloc函数中是先找到一个可用块，再进行处理的，答案是统一在一起进行处理的。另外，主要参考了答案的le2page的用法，对链表和页的关系有了更深的认识。

##[练习2]  

###实验内容：通过设置页表和对应的页表项， 可建立虚拟内存地址和物理内存地址的对应关系。 其中的get_pte函数是设置页表项环节 中的一个重要步骤。 此函数找到一个虚地址对应的二级页表项的内核虚地址， 如果此二级页表项不存在， 则分配一个包含此项的二级页表。 需要补全get_pte函数 in kern/mm/pmm.c， 实现其功能。请在实验报告中简 要说明你的设计实现过程。 

答：

> getpte功能：给定一个虚拟地址，返回此虚拟地址在二级页表中对应的项，实现：   
>1） 通过线性地址找到 一级页表项；  
>2） 如果在查找二级页表项时， 发现对应的二级页表不存在， 则需要根据create参数的值来处理是否创建新的二级页表。  如果create参数不为0， 则get_pte需要申请一个新的物理页,再在一级页表中添加页目。  
> 3）初始化对应的页地址 。新申请的页必
须全部设定为零（利用memset函数）， 因为这个页所代表的虚拟地址都没有被映射。   
> 4）设置页的标志位    
> 5）返回地址。里面主要用到的定义：  
> >&((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]; 
 

### 请描述页目 录项（Pag Director Entry） 和页表（Page Table Entry） 中每个组成部分的含义和以及对ucore而言的潜在用处。
答：包括地址和一些标志位，标志位在mmu.h的文件中有对PDE和PTE进行了定义：

	/* page table/directory entry flags */
	#define PTE_P           0x001                   // Present
	#define PTE_W           0x002                   // Writeable
	#define PTE_U           0x004                   // User
	#define PTE_PWT         0x008                   // Write-Through
	#define PTE_PCD         0x010                   // Cache-Disable
	#define PTE_A           0x020                   // Accessed
	#define PTE_D           0x040                   // Dirty
	#define PTE_PS          0x080                   // Page Size
	#define PTE_MBZ         0x180                   // Bits must be zero
	#define PTE_AVAIL       0xE00                   // Available for software use
	                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
	                                                // hardware, so user processes are allowed to set them arbitrarily.

第一位标志是否缺页。第二位表示是否可写。第三位判断是否用户态可访问。第四位表示是否写直达位。第五位标志Cache是否开启。第六位记录访问情况，用于需要替换时判断是否最近有用。第七位标志有没有被修改，确定是否需要写回外存。第8-9位是保留位。第10-12位是可用位，供给用户态软件使用。

###如果ucore执行过程中访问 内存， 出现了页访问 异常， 请问 硬件要做哪些事情？
发生中断，操作系统对外存进行读写，置换内存内容等处理

##练习三：释放某虚地址所在的页并取消对应二级页表项的映射
###设计实现
>1)检查这个pte是否有效；  
>2）若有效，查询到它对应的页。  
>3）将它对应的页关联数减一。  
>4）如果该页的关联数为0，那么释放该页。  
>5）将该pte置为0；  
>6）修改TLB。

###和标准答案的差别

差别几乎没有。。。

###数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

```
//virtual address of physicall page array  
struct Page *pages; 
...
pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);
```
看起来pages像是在BSS段结束之后申请的一片内存，所以pages应该是那一片内存的虚拟地址
###如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？
在kern/mm/memlayout.h里有一个变量 kernbase,将0XC0000000改为0。


