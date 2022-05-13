### Trap机制

trap完成用户空间和内核空间的切换。

32个用户寄存器+stack pointer（堆栈寄存器）

![image-20220505105634644](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505105634644.png)

![image-20220505105816014](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505105816014.png)

supervised mode 能干什么？

![image-20220505110019978](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505110019978.png)



ECALL指令会切换到具有supervisor mode的内核中。在这个过程中，内核中执行的第一个指令是一个由汇编语言写的函数，叫做uservec。这个函数是内核代码trampoline.s文件的一部分。所以执行的第一个代码就是这个uservec汇编函数。

在这个汇编函数中，代码执行跳转到了由C语言实现的函数usertrap中，这个函数在trap.c中。

在usertrap这个C函数中，我们执行了一个叫做syscall的函数。

这个函数会在一个表单中，根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505110654798.png" alt="image-20220505110654798" style="zoom:50%;" />

![image-20220505110923120](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505110923120.png)

STVEC寄存器是一个只能在supervisor mode下读写的特权寄存器。在从内核空间进入到用户空间之前，内核会设置好STVEC寄存器指向内核希望trap代码运行的位置。

ecall之前会将系统调用号放进寄存器

### ecall

ecall做的事：

ecall从user mode改为supervised mode，将PC保存在SEPC寄存器，将STVEC加载到PC（STVEC是trampoline page中的地址，指向uservec）

ecall只会做上面三件事。

![image-20220505200913032](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505200913032.png)



### uservec

XV6在每个user page table映射了trapframe page，这样每个进程都有自己的trapframe page。可以有一块内存用来保存信息（0x3ffffffe000）。

有一个寄存器SSCRATCH保存了这里的地址（0x3ffffffe000）。

然后就是保存32个寄存器。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505202226643.png" alt="image-20220505202226643" style="zoom:50%;" />

这一行的作用是将kernel stack pointer 加载到esp寄存器。

然后把kernel page table的地址加载到SATP寄存器（这个寄存器在用户态指向user page table，在内核态指向 kernel page table）

![image-20220505203123132](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220505203123132.png)

然后进入usertrap函数。



### usertrap

查看scause寄存器，如果是8，代表系统调用，调用syscall函数



### syscall

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507084327578.png" alt="image-20220507084327578" style="zoom: 50%;" />

根据a7寄存器的值查找对应的系统调用函数(sys_...)，然后进行系统调用。

sys_...函数会去处理函数传入的参数，然后跳转到C语言的函数的地方执行。



### usertrapret

trap返回 

设置stvec指向trampoline代码，在那里最终会执行sret指令返回到用户空间。

存储kernel page table的指针，存储当前用户进程的kernel stack，存储了usertrap函数的指针，这样trampoline代码才能跳转到这个函数



### userret

程序会切换回user mode，SEPC寄存器的数值会被拷贝到PC寄存器（程序计数器），重新打开中断

