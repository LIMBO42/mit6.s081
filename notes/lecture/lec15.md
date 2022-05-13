### File System Crash

![image-20220512150147449](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512150147449.png)

在这个位置，我们先写了block 33表明inode已被使用，之后出现了电力故障，然后计算机又重启了。这时，我们丢失了刚刚分配给文件x的inode。这个inode虽然被标记为已被分配，但是它并没有放到任何目录中，所以也就没有出现在任何目录中，因此我们也就没办法删除这个inode。



#### File System logging

![image-20220512150420458](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512150420458.png)

（log write）当需要更新文件系统时，我们并不是更新文件系统本身。假设我们在内存中缓存了bitmap block，也就是block 45。当需要更新bitmap时，我们并不是直接写block 45，而是将数据写入到log中，并记录这个更新应该写入到block 45。对于所有的写 block都会有相同的操作，例如更新inode，也会记录一条写block 33的log。**（写数据不是真的写入，而是先写入log block）**

（commit op）之后在某个时间，当文件系统的操作结束了，比如说我们前一节看到的4-5个写block操作都结束，并且都存在于log中，我们会commit文件系统的操作。这意味着我们需要在log的某个位置记录属于同一个文件系统的操作的个数，例如5。

（install log）当我们在log中存储了所有写block的内容时，如果我们要真正执行这些操作，只需要将block从log分区移到文件系统分区。我们知道第一个操作该写入到block 45，我们会直接将数据从log写到block45，第二个操作该写入到block 33，我们会将它写入到block 33，依次类推。

（clean log）一旦完成了，就可以清除log。清除log实际上就是将属于同一个文件系统的操作的个数设置为0。

如果crash了，在重启的时候，文件系统会查看log的commit记录值，如果是0的话，那么什么也不做。如果大于0的话，我们就知道log中存储的block需要被写入到文件系统中。很明显我们在crash的时候并不一定完成了install log，我们可能是在commit之后，clean log之前crash的。所以这个时候我们需要做的就是reinstall（注，也就是将log中的block再次写入到文件系统），再clean log。

1. log write
2. commit log
3. install log
4. clean log

1-2之间crash，没有问题，写入了log block，相当于没写入，系统调用没有发生，因为操作的数目=0；

2-3之间crash，提交了log，可能正在install的时候crash，恢复的时候假装没有install，因为重复写入没有问题。

3-4之间crash，一样的，重复写入即可。



这里的commit log的写入可以认为是原子操作。

**这样就使得一系列的写入像原子操作一样。**

前提：协议都遵循了write ahead rule，也就是说在写入commit记录之前，你需要确保所有的**写操作都在log中**。

