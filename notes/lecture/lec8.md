page fault和其他的异常使用与系统调用相同的trap机制来从用户空间切换到内核空间。

所以，从硬件和XV6的角度来说，当出现了page fault，现在有了3个对我们来说极其有价值的信息，分别是：

- 引起page fault的内存地址
- 引起page fault的原因类型
- 引起page fault时的程序计数器值，这表明了page fault在用户空间发生的位置

sbrk是XV6提供的系统调用，它使得用户应用程序能扩大自己的heap。

#### lazy allocation

核心思想非常简单，sbrk系统调基本上不做任何事情，唯一需要做的事情就是提升*p->sz*，将*p->sz*增加n，其中n是需要新分配的内存page数量。但是内核在这个时间点并不会分配任何物理内存。之后在某个时间点，应用程序使用到了新申请的那部分内存，这时会触发page fault，因为我们还没有将新的内存映射到page table。所以，如果我们解析一个大于旧的*p->sz*，但是又小于新的*p->sz（注，也就是旧的p->sz + n）*的虚拟地址，我们希望内核能够分配一个内存page，并且重新执行指令。通过kalloc函数分配一个内存page；初始化这个page内容为0；将这个内存page映射到user page table中；最后重新执行指令。

在我们为您提供的内核中，sbrk() 分配物理内存并将其映射到进程的虚拟地址空间。但是如果用户请求的内存很多，内核为其分配和映射内存可能需要很长时间。此外，一些程序分配的内存常常比它们实际使用的多（例如，为了实现稀疏数组），或者在使用之前就分配了内存。为了让 sbrk() 在这些情况下更快地完成，复杂的内核会延迟分配。也就是说， sbrk() 没有真正分配物理内存，其只是记住了要分配给哪些用户虚拟地址，并在用户页表中将这些地址标记为无效。当进程第一次尝试使用这些地址时，CPU 会产生page fault并trap进内核，内核此时才真正分配物理内存，并将其memset为0和在用户页表中添加映射来处理该page fault。您将在本实验中将此惰性分配功能添加到 xv6。

![image-20220507150853753](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507150853753.png)

r_scause = 15 为 store page fault，出现了向未被分配的内存写入数据的情况。

在上面增加的代码中，首先打印一些调试信息。之后分配一个物理内存page，如果ka等于0，表明没有物理内存我们现在OOM了，我们会杀掉进程。如果有物理内存，首先会将内存内容设置为0，之后将物理内存page指向用户地址空间中合适的虚拟内存地址。

![image-20220507151040924](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507151040924.png)

实际上并没有正常工作。我们这里有两个page fault，第一个对应的虚拟内存地址是0x4008，但是很明显在处理这个page fault时，我们又有了另一个page fault 0x13f48。现在唯一的问题是，uvmunmap在报错，一些它尝试unmap的page并不存在。这里unmap的内存是什么？

***学生回答：之前lazy allocation但是又没有实际分配的内存。***

之前的panic表明，我们尝试在释放一个并没有map的page。怎么会发生这种情况呢？唯一的原因是sbrk增加了p->sz，但是应用程序还没有使用那部分内存。因为对应的物理内存还没有分配，所以这部分新增加的内存的确没有映射关系。

在uvmunmap中直接continue即可。



#### zero-fill-on-demand

text区域，data区域，同时还有一个**BSS区域**

例如你在C语言中定义了一个大的矩阵作为全局变量，它的元素初始值都是0，为什么要为这个矩阵分配内存呢？其实只需要记住这个矩阵的内容是0就行。

在一个正常的操作系统中，如果执行exec，exec会申请地址空间，里面会存放text和data。因为BSS里面保存了未被初始化的全局变量，这里或许有许多许多个page，但是所有的page内容都为0。

通常可以调优的地方是，我有如此多的内容全是0的page，在物理内存中，我只需要分配一个page，这个page的内容全是0。然后将所有虚拟地址空间的全0的page都map到这一个物理page上。这样至少在程序启动的时候能节省大量的物理内存分配。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507151407571.png" alt="image-20220507151407571" style="zoom:67%;" />

当然这里的mapping需要非常的小心，我们不能允许对于这个page执行写操作，因为所有的虚拟地址空间page都期望page的内容是全0，所以这里的PTE都是只读的。之后在某个时间点，应用程序尝试写BSS中的一个page时，比如说需要更改一两个变量的值，我们会得到page fault。然后创建一个新的page，将其内容设置为0，并重新执行指令。



#### copy on write

当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存page。

![image-20220507181700077](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507181700077.png)

parent和child指向了同一块物理页表

在某个时间点，当我们需要更改内存的内容时，我们会得到page fault。因为父进程和子进程都会继续运行，而父进程或者子进程都可能会执行store指令来更新一些全局变量，这时就会触发page fault，因为现在在向一个只读的PTE写数据。



#### demand paging

所以对于exec，在虚拟地址空间中，我们为text和data分配好地址段，但是相应的PTE并不对应任何物理内存page。对于这些PTE，我们只需要将valid bit位设置为0即可。

应用程序是从地址0开始运行。text区域从地址0开始向上增长。位于地址0的指令是会触发第一个page fault的指令，因为我们还没有真正的加载内存。

那么该如何处理这里的page fault呢？首先我们可以发现，这些page是on-demand page。我们需要在某个地方记录了这些page对应的程序文件，我们在page fault handler中需要从程序文件中读取page数据，加载到内存中；之后将内存page映射到page table；最后再重新执行指令。



#### Memory map files

将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件。

mmap(va, len, protection, flags, fd, off)

将fd对应的文件的off处len长度的内容映射到虚拟地址va，这样就可以利用load/store来操作文件。

处理结束之后unmap即可

