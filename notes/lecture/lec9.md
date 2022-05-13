![image-20220508122916985](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508122916985.png)

![image-20220508143158951](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508143158951.png)

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508145714275.png" alt="image-20220508145714275" style="zoom:50%;" />

![image-20220508145724655](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508145724655.png)

Shell输出的每一个字符都会触发一个write系统调用。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508145802770.png" alt="image-20220508145802770" style="zoom:50%;" />

在filewrite函数中首先会判断文件描述符的类型。mknod生成的文件描述符属于设备（FD_DEVICE），而对于设备类型的文件描述符，我们会为这个特定的设备执行设备相应的write函数。因为我们现在的设备是Console，所以我们知道这里会调用console.c中的consolewrite函数。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508145908900.png" alt="image-20220508145908900" style="zoom:50%;" />

uartputc函数会稍微有趣一些。在UART的内部会有一个buffer用来发送数据，buffer的大小是32个字符。同时还有一个为consumer提供的读指针和为producer提供的写指针，来构建一个环形的buffer（注，或者可以认为是环形队列）。

![image-20220508145957024](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220508145957024.png)

如果满了，sleep，将cpu让给其他进程。否则写入buffer，uartstart就是通知设备执行操作。首先是检查当前设备是否空闲，如果空闲的话，我们会从buffer中读出数据，然后将数据写入到THR（Transmission Holding Register）发送寄存器。这里相当于告诉设备，我这里有一个字节需要你来发送。一旦数据送到了设备，系统调用会返回，用户应用程序Shell就可以继续执行。



中断处理：

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNfH5qMvmyxhegTFSUo%2F-MNpbLBt88M7WKDIGi4a%2Fimage.png?alt=media&token=6b9561db-5941-4caa-8187-809dae89a377)

