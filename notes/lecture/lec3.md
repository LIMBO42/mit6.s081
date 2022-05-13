#### 页表：

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKKjB2an4WcuUmOlE__%2F-MKOK5mRXs0VPRGM1JUF%2Fimage.png?alt=media&token=86e8e56f-a87e-4826-b77d-ff37248595e5)

cpu取指令--->mmu(memory management unit)--->mem

mmu中有一张表记录虚拟地址和物理地址的对应关系；同时有寄存器(SATP)记录这张表在物理内存中的基地址。

这里的基本想法是**每个应用程序都有自己独立的表单**，并且这个表单定义了应用程序的地址空间。所以当操作系统将CPU从一个应用程序切换到另一个应用程序时，同时也需要切换SATP寄存器中的内容。

内核会写SATP寄存器，写SATP寄存器是一条特殊权限指令。所以，用户应用程序不能通过更新这个寄存器来更换一个地址对应表单，否则的话就会破坏隔离性。所以，只有运行在kernel mode的代码可以更新这个寄存器。

![image-20220428142331491](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220428142331491.png)

TLB提供了虚拟地址转换成物理地址的缓存

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220501092130522.png" alt="image-20220501092130522" style="zoom: 50%;" />

有一些page在虚拟内存中的地址很靠后，比如kernel stack在虚拟内存中的地址就很靠后。这是因为在它之下有一个未被映射的Guard page，这个Guard page对应的PTE的Valid 标志位没有设置，这样，如果kernel stack耗尽了，它会溢出到Guard page，但是因为Guard page的PTE中Valid标志位未设置，会导致立即触发page fault，这样的结果好过内存越界之后造成的数据混乱。立即触发一个panic（也就是page fault），你就知道kernel stack出错了。同时我们也又不想浪费物理内存给Guard page，所以Guard page不会映射到任何物理内存，它只是占据了虚拟地址空间的一段靠后的地址。

同时，kernel stack被映射了两次，在靠后的虚拟地址映射了一次，在PHYSTOP下的Kernel data中又映射了一次，但是实际使用的时候用的是上面的部分，因为有Guard page会更加安全。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220501092235314.png" alt="image-20220501092235314" style="zoom: 50%;" />

RWX标志位控制读写执行权限

<img src="C:\Users\LIMBO\AppData\Roaming\Typora\typora-user-images\image-20220501092612034.png" alt="image-20220501092612034" style="zoom:50%;" />

为什么有多个kernel stack？每个进程都有一个kernel stack和一个user stack。



精简指令集和复杂指令集

RSICV是精简指令集，X86是复杂指令集。

根据数目的多少区分精简和复杂，为什么复杂，因为一条指令做了很多事情。

高通的骁龙处理器就是精简指令集ARM，包括M1芯片，也采用ARM架构。



![image-20220503123413318](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220503123413318.png)

函数调用过程中可不可能发生改变



栈帧

从高地址向低地址增长（向下）

![image-20220503123644525](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220503123644525.png)

![image-20220503123751244](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220503123751244.png)

**（和x86的函数调用的栈帧不一样）**

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220503123905874.png" alt="image-20220503123905874" style="zoom:50%;" />

