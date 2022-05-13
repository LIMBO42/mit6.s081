Operating system interface：

shell是用户态的程序：从用户那读取输入然后执行。

 

### Process and memory

The exec system call replaces the calling process’s memory with a new memory image loaded from a file stored in the file system. 



### Pipes

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220421201137429.png" alt="image-20220421201137429" style="zoom:67%;" />

这里实现了往pipe里面写，然后wc统计的功能。

question：这里为什么不需要wait来同步？因为pipe空，进程会阻塞住。

read要么等待读，要么pipe的所有写端均被关闭（所以子进程要先关闭自己的p[1]，然后在exec，否则由于p[1]没有全部被关闭，会一直阻塞）。



比较pipe和临时文件：

```bash
echo hello world | wc
echo hello world >/tmp/xyz; wc </tmp/xyz
```

pipe的优点：不需要处理掉临时文件，也就不需要磁盘空间；可以并发执行管道的左右；进程间的通信的时候，阻塞的语义相比临时文件频繁的读写要方便很多。



### File System

inode可以有多个names，这些names称为links，link由dictionary中的一条entry构成，entry有name和指向inode的reference。

inode包含了文件的元数据，type，length，location of the file，and the number of links

![image-20220422081353974](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220422081353974.png)

![image-20220422081417425](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220422081417425.png)

POSIX 规范 portabale operating system interface 接口的规范





xv6中的系统调用的说明：

![image-20220422183109016](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220422183109016.png)
