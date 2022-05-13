#### File System



![image-20220512091313109](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512091313109.png)



<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512091440851.png" alt="image-20220512091440851" style="zoom:67%;" />

The disk layer reads and writes blocks on an virtio hard drive. The buffer cache layer caches disk blocks and synchronizes access to them, making sure that only one kernel process at a time can modify the data stored in any particular block. The logging layer allows higher layers to wrap updates to several blocks in a transaction, and ensures that the blocks are updated atomically in the face of crashes (i.e., all of them are updated or none). The inode layer provides individual files, each represented as an inode with a unique i-number and some blocks holding the file’s data. The directory layer implements each directory as a special kind of inode whose content is a sequence of directory entries, each of which contains a file’s name and i-number. The pathname layer provides hierarchical path names like /usr/rtm/xv6/fs.c, and resolves them with recursive lookup. The file descriptor layer abstracts many Unix resources (e.g., pipes, devices, files, etc.) using the file system interface, simplifying the lives of application programmers.

buffer cache 保证了文件的access的同步，logging层保证了事务原子性，inode层将文件抽象成inode，directory也是inode。



#### Disk

Disk hardware traditionally presents the data on the disk as a numbered sequence of 512-byte blocks (also called sectors): sector 0 is the first 512 bytes, sector 1 is the next, and so on. Xv6 holds copies of blocks that it has read into memory in objects of type struct buf (kernel/buf.h:1).

![image-20220512091941443](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512091941443.png)

block 0里面是boot，block 1 是超级块。Blocks starting at 2 hold the log. After the log are the inodes, with multiple inodes per block. After those come bitmap blocks tracking which data blocks are in use. The remaining blocks are data blocks; each is either marked free in the bitmap block, or holds content for a file or directory. The superblock is filled in by a separate program, called mkfs, which builds an initial file system.

#### Buffer cache

(1) synchronize access to disk blocks to ensure that only one copy of a block is in memory and that only one kernel thread at a time uses that copy; (2) cache popular blocks so that they don’t need to be re-read from the slow disk.

起同步和cache的作用

![image-20220512092417900](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512092417900.png)

从disk读进buf，buffer cache is a doubly-linked list of buffers.

Buffer cache层给上层提供的主要接口是`bread`和`bwrite`；bread获取一个struct buf，其中存储一个缓存在内存中的可读写的磁盘块副本，bwrite将修改后的缓存块写入磁盘上的相应块。内核线程必须通过调用`brelse`释放bread读的块。Buffer cache为每个struct buf使用一个睡眠锁，以确保每个struct buf（因此也是每个磁盘块）每次只被一个线程使用；`bread`返回一个上锁的struct buf，`brelse`释放该锁。



#### logging

解决事务问题，原子性



Xv6通过简单的日志记录形式解决了文件系统操作期间的崩溃问题。xv6系统调用**不会首先直接写入磁盘上的文件系统数据结构**。相反，它会在磁盘上的*log*中放置它希望进行的所有磁盘写入的**描述**。一旦系统调用记录了它的所有写入操作，它就会向磁盘写入一条特殊的***commit\***（提交）记录，表明日志包含一个完整的操作。此时，系统调用才会对磁盘上的文件系统数据结构进行写入。确实完成这些写入后，系统调用将擦除磁盘上的日志。

如果系统崩溃并重新启动，则在运行任何进程之前，文件系统代码将按如下方式从崩溃中恢复。如果日志标记为完整的操作（含有*commit）*，则恢复代码会在磁盘文件系统中的对应位置执行这些写操作（redo）。如果日志没有标记为包含完整操作（不含*commit）*，则恢复代码将忽略该日志（undo）。之后恢复代码将会擦除所有的这些日志。

为什么xv6的日志解决了文件系统操作期间的崩溃问题？如果崩溃发生在操作提交之前，那么磁盘上的日记将不会被标记为已完成，恢复代码将忽略它，并且磁盘的状态将如同操作完全没有执行一样。如果崩溃发生在操作提交之后，则恢复代码将复现所有写入操作，如果之前系统调用已经开始写入磁盘数据结构，则可能会重复这些操作。在任何一种情况下，日志都会**使操作在崩溃时成为原子操作**：恢复后，要么操作的所有写入都显示在磁盘上，要么都不显示。

**总之，日志的作用是保证一系列对磁盘的更新操作的原子性，使其成为一个事务，防止操作进行了一部分后停止使文件系统进入不一致状态。**

#### Inode

![image-20220513143733501](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220513143733501.png)

两层意思：磁盘上的和内存里的

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220513143848526.png" alt="image-20220513143848526" style="zoom:67%;" />



<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220513144034098.png" alt="image-20220513144034098" style="zoom:67%;" />

addrs是文件块的索引

on-disk的inode是连续的

The type field distinguishes between files, directories, and special files (devices).

可以添加文件类型，比如实验中可以添加文件为符号链接

 A type of zero indicates that an on-disk inode is free. The nlink field counts the number of directory entries that refer to this inode, in order to recognize when the on-disk inode and its data blocks should be freed. The size field records the number of bytes of content in the file. The addrs array records the block numbers of the disk blocks holding the file’s content.



The kernel **keeps the set of active inodes in memory in a table called itable;** The kernel stores an inode in memory only if there are C pointers referring to that inode. 

应该是打开文件的意思，ref记录了索引的多少

 The ref field **counts the number of C pointers** referring to the in-memory inode, and the kernel discards the inode from memory if the reference count drops to zero.  Pointers to an inode can come from file descriptors, current working directories, and transient kernel code such as exec.



##### directionory

类似文件，inode的type为dir，并且数据是一系列的directory entries。Each entry is a struct dirent (kernel/fs.h:56), which contains a name and an inode number. 

也就是说建立了文件名和inode的映射



#### FILE descriptor

A cool aspect of the Unix interface is that most resources in Unix are represented as files, including devices such as the console, pipes, and of course, real files. The file descriptor layer is the layer that achieves this uniformity.

通过文件描述符实现了文件级别的抽象

每个进程都有打开文件表，文件用struct File 表示，which is a wrapper around either an inode or a pipe, plus an I/O offset.

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220513150907670.png" alt="image-20220513150907670" style="zoom:67%;" />

 Each call to open creates a new open file (a new struct file): if multiple processes open the same file independently, the different instances will have different I/O offsets. 

不同的进程有自己的File，维护了各自的off

 On the other hand, a single open file (the same struct file) can appear multiple times in one process’s file table and also in the file tables of multiple processes.

**All the open files in the system are kept in a global file table**, the ftable. The file table has functions to allocate a file (filealloc), create a duplicate reference (filedup), release a reference (fileclose), and read and write data (fileread and filewrite).

The first three follow the now-familiar form. Filealloc (kernel/file.c:30) scans the file table for an unreferenced file (f->ref == 0) and returns a new reference; filedup (kernel/file.c:48) increments the reference count; and fileclose (kernel/file.c:60) decrements it. When a file’s reference count reaches zero, fileclose releases the underlying pipe or inode, according to the type.

##### sys_link

sys_link (kernel/sysfile.c:120) begins by fetching its arguments, two strings old and new (kernel/sysfile.c:125). Assuming old exists and is not a directory (kernel/sysfile.c:129-132), sys_link increments its ip->nlink count. Then sys_link calls nameiparent to find the parent directory and final path element of new (kernel/sysfile.c:145) and creates a new directory entry pointing at old ’s inode (kernel/sysfile.c:148). The new parent directory must exist and be on the same device as the existing inode: inode numbers only have a unique meaning on a single disk.

硬链接：在new的directory中添加一个entry，指向old的inode

##### create

The function create (kernel/sysfile.c:242) creates a new name for a new inode. It is a generalization of the three file creation system calls: open with the O_CREATE flag makes a new ordinary file, mkdir makes a new directory, and mkdev makes a new device file. Like sys_link, create starts by calling nameiparent to get the inode of the parent directory. It then calls dirlookup to check whether the name already exists (kernel/sysfile.c:252). If the name does exist, create’s behavior depends on which system call it is being used for: open has different semantics from mkdir and mkdev. If create is being used on behalf of open (type == T_FILE) and the name that exists is itself a regular file, then open treats that as a success, so create does too (kernel/sysfile.c:256). Otherwise, it is an error (kernel/sysfile.c:257-258). If the name does not already exist, create now allocates a new inode with ialloc (kernel/sysfile.c:261). If the new inode is a directory, create initializes it with . and .. entries. Finally, now that the data is initialized properly, create can link it into the parent directory (kernel/sysfile.c:274).