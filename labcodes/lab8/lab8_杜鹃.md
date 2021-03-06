# 操作系统Lab8实验报告

2012011354
计24
杜鹃

## 练习零：填写已有实验
使用understand的merge功能进行merge  


## 练习 1: 完成读文件操作的实现（需要编码）


###首先了解打开文件的处理流程， 然后参考本实验后续的文件读写操作的过程分析， 编写在sfs_inode.c中sfs_io_nolock读文件中数据的实现代码
设计：   
在sfs_io_nolock函数中， 有sfs_buf_op = sfs_rbuf,sfs_block_op = sfs_rblock， 设置读取的函数操作。 接着进行实际操作， 先处理起始的没有对齐到块的部分， 再以块为单位循环处理中间 的部分， 最后处理末尾剩余的部分。 每部分中都调用sfs_bmap_load_nolock函数得到blkno对应的inode编号， 并调用sfs_rbuf或sfs_rblock函数读取数据（中间 部分调用sfs_rblock， 起始和末尾部分调用sfs_rbuf） ， 调整相关变量。 完成后如果offset + alen > din->fileinfo.size（写文件时会出现这种情况， 读文件时不会出现这种情况， alen为实际读写
的长度） ， 则调整文件大小为offset + alen并设置dirty变量


###请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案， 鼓励给出详细设计方案
使用VFS机制提供的接口，设计一个pipe用的pipefs,在系统调用时，创建出两个file，一个只读，一个只写，并使这两个file连接到同一个inode上，inode里使用pipefs里定义的pipefs_inode，并在其中维护一个数据缓冲区，使用VFS提供的read/write接口，在该数据缓冲区里读写数据。

在缓冲区里提供了一个head和一个tail指针，当两个指针重合时数据为空，并且在filefs_inode中提供了一个管程wait,用来对进程间的同步进行管理。

写数据的过程如下：首先进入管程，当有数据要写入时，不断循环，循环中的内容为，首先获取当前缓冲区里的空闲空间，如果没有空闲空间，则使用管程通知当前睡在条件变量上的进程进行读操作，如果没有需要进行读操作的进程，则直接返回，否则当前进程睡眠，等待其他进程读数据。当缓冲区中有空闲空间时，将数据写入缓冲区中，并同步更新指针，已写字节数，以及剩余字节数。

## 练习二：完成基于文件系统的执行程序机制的实现（需要编码）


### 请在实验报告中给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案， 鼓励给出详细设计方案
(1)硬链接

硬链接值多个文件名指向同一索引节点。在Linux中，硬连接的作用是允许一个文件拥有多个有效路径名，这样就可以建立硬连接到重要文件，以防止“误删”的功能。建立硬链接后，只删除一个连接并不影响索引节点本身和其它的连接，只有当最后一个连接被删除后，文件的数据块及目录的连接才会被释放。

在lab8中，打开一个文件或直接从内核用do_fork加载一个程序，对应的函数调用链如下：
```
    sys_exec->do_execve->sysfile_open->file_open->vfs_open->vfs_lookup->vop_lookup(sfs_lookup)->sfs_lookup_once-        >sfs_load_inode->sfs_create_inode->vop_init(inode_init)
```    
在Linux中，每个inode有一个inode节点编号，将f2硬链接到f1后，f2和f1的inode节点编号相同，即二者对应的是同一个inode。在上面的函数调用链中，file_open函数中，新建了inode指针，然后在SFS层进行具体的文件inode创建。大概思路是，可以增加一个创建硬链接的函数，对某个path创建一个inode指针，然后获取要链接到的文件的inode，直接用这个inode的地址对path的inode*赋值，这样就保证创建硬链接的path和被链接的file的inode相同。然后还需要将对应的sfs_disk_inode的nlinks加一。lab8中并未涉及文件的删除，如果要增加文件的删除功能，则需要对某个文件inode的硬链接数进行判断，以确定是否最后删除。

(2)软链接

软链接类似windows中的快捷方式，在linux里，软链接是一个文本文件，其中的内容是被链接的文件的位置信息。因为软链接是一个新的文件，所以要实现软链接，首先要实现创建文件的功能。根据path(或name)创建一个文件，然后新建inode和进行相关的初始化。openfile时，先检查文件是否存在，未存在则新创建。新增一个创建软链接的函数，参数是被链接的文件和作为软连接(快捷方式)的文件，获取这两个文件的inode，如果文件不存在则创建。然后将被链接文件inode中存储的相关信息，尤其是位置信息写入软链接文件。此外，需要实现的是打开软链接文件实际上是打开被链接的文件,可以通过修改sfs inode，在其中增加成员变量表示是否是软链接，读取时则直接根据inode里的标识来进行操作。

## 重要知识点
了解基本的文件系统系统调用的实现方法；  
了解一个基于索引节 点组织方式的Simple FS文件系统的设计与实现；  
了解文件系统抽象层 -VFS的设计与实现；
