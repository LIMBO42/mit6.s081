![image-20220426185641239](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220426185641239.png)



<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220426190102732.png" alt="image-20220426190102732" style="zoom:67%;" />

 Instructions come first, followed by global variables, then the stack, and finally a “heap” area (for malloc) that the process can expand as needed



在page tables上查询虚拟地址的时候，硬件只使用低39位，而xv6使用38位，因此地址空间为$2^{38} − 1 = 0x3fffffffff$ ，xv6地址空间的最高是a page for *trampoline*，and a page mappi*ng the progress‘s trapframe*。

Xv6 uses these two pages to transition into the kernel and back; 

the trampoline page contains the code to transition in and out of the kernel and mapping the trapframe is necessary to save/restore the state of the user process。

进程的状态保存在 struct proc中， A process’s most important pieces of kernel state are its page table, its kernel stack, and its run state.

每个进程都有两个stack，user stack和kernel stack。在用户态使用user stack，kernel stack 为空，而在内核态使用kernel stack，user stack保存saved data (local variables, function call return addresses)

**两种抽象**：

a process bundles two design ideas: an address space to give a process the illusion of its own memory, and, a thread, to give the process the illusion of its own CPU. 

内存空间的抽象和cpu的抽象。



#### XV6启动过程：

1.RISCV power on的时候，先运行ROM上的boot loader，会将xv6 kernel加载进入内存，loader把kernel加载到$0x80000000$处。

2.然后CPU从entry.s中的*_entry*开始执行，此时是machine mode（没有分页，虚拟地址就是物理地址）

3.*_entry*处的指令会建立stack0给C code 使用（将sp寄存器指向esp+4096，栈从高向低增长），然后执行start.c中的程序。

4.start会做必要的设置，使os从machine mode 转向supervised mode（保护模式），另外会开启时钟中断，This causes the program counter to change to main (kernel/main.c:11)。start将main的地址放进寄存器，通过mret切换到supervised mode，并跳转到main执行。

5.mian函数：初始化设置，然后通过userinit()启动第一个用户进程：userinit对应以下汇编程序，执行系统调用exec。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220427092443028.png" alt="image-20220427092443028" style="zoom:67%;" />

6.传入的$a1对应argv，argv[0]为exec执行的文件的路径："/init\"。因此，initcode完成了通过exec调用init程序。

7.Init (user/init.c:15) creates a new console device file if needed and then opens it as file descriptors 0, 1, and 2. Then it starts a shell on the console. The system is up.

