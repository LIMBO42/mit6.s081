There are three kinds of event which cause the CPU to set aside ordinary execution of instructions and force a transfer of control to special code that handles the event. One situation is a system call, when a user program executes the ecall instruction to ask the kernel to do something for it. Another situation is an exception: an instruction (user or kernel) does something illegal, such as divide by zero or use an invalid virtual address. The third situation is a device interrupt, when a device signals that it needs attention, for example when the disk hardware finishes a read or write request.

The usual sequence is that a trap forces a transfer of control into the kernel; the kernel saves registers and other state so that execution can be resumed; the kernel executes appropriate handler code (e.g., a system call implementation or device driver); the kernel restores the saved state and returns from the trap; and the original code resumes where it left off.

中断处理程序/中断向量

![image-20220507092536963](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220507092536963.png)

具体过程可以看书



copyin，copyout：Since pagetable is not the current page table, copyinstr uses walkaddr (which calls walk) to look up srcva in pagetable, yielding physical address pa0. The kernel maps each physical RAM address to the corresponding kernel virtual address, so copyinstr can directly copy string bytes from pa0 to dst.

copyin是将用户虚拟地址src上的内容copy到kernel中的dest中，

copyout将kernel中src的内容copy到虚拟地址dest上。

因为内核那边的虚拟地址和物理地址一一映射，所以内核的地址不需要转化







### page-fault Copy on write

use page faults to implement copy-on-write (COW) fork.

fork causes the child’s initial memory content to be the same as the parent’s at the time of the fork.

xv6 implements fork with uvmcopy (kernel/vm.c:301), which allocates physical memory for the child and copies the parent’s memory into it. It would be more efficient if the child and parent could share the parent’s physical memory.

xv6是将copy一份父进程的页面给子进程，但如果能够share就能提高效率，在需要写的时候再copy。

The CPU raises a page-fault exception when a virtual address is used that has no mapping in the page table, or has a mapping whose PTE_V flag is clear, or a mapping whose permission bits (PTE_R, PTE_W, PTE_X, PTE_U) forbid the operation being attempted. 

page table中的虚拟地址到物理地址的映射没有或失效，此时有page-fault。

The basic plan in COW fork is for the parent and child to initially share all physical pages, but for each to map them read-only (with the PTE_W flag clear).

Parent and child can read from the shared physical memory. If either writes a given page, the RISC-V CPU raises a page-fault exception. 

 The kernel’s trap handler responds by allocating a new page of physical memory and copying into it the physical page that the faulted address maps to. 

 The kernel changes the relevant PTE in the faulting process’s page table to point to the copy and to allow writes as well as reads, and then resumes the faulting process at the instruction that caused the fault.



### lazy-allocation

First, when an application asks for more memory by calling sbrk, the kernel notes the increase in size, but does not allocate physical memory and does not create PTEs for the new range of virtual addresses. Second, on a page fault on one of those new addresses, the kernel allocates a page of physical memory and maps it into the page table.

sbrk申请内存空间，但并不会实际分配，在page fault的时候才会真正分配。

Since applications often ask for more memory than they need, lazy allocation is a win: the kernel doesn’t have to do any work at all for pages that the application never uses. Furthermore, if the application is asking to grow the address space by a lot, then sbrk without lazy allocation is expensive: if an application asks for a gigabyte of memory, the kernel has to allocate and zero 262,144 4096-byte pages. Lazy allocation allows this cost to be spread over time.  On the other hand, lazy allocation incurs the extra overhead of page faults, which involve a kernel/user transition. Operating systems can reduce this cost by allocating a batch of consecutive pages per page fault instead of one page and by specializing the kernel entry/exit code for such page-faults.



#### demand paging

In exec, xv6 loads all text and data of an application eagerly into memory. Since applications can be large and reading from disk is expensive, this startup cost may be noticeable to users: when the user starts a large application from the shell, it may take a long time before user sees a response. To improve response time, a modern kernel creates the page table for the user address space, but marks the PTEs for the pages invalid. On a page fault, the kernel reads the content of the page from disk and maps it into the user address space. Like COW fork and lazy allocation, the kernel can implement this feature transparently to applications.

执行的时候不把所有的text和data加载，而是分配PTE，不过将有效位置为0，需要的时候再去读取。



#### paging to disk

The idea is to store only a fraction of user pages in RAM, and to store the rest on disk in a paging area. The kernel marks PTEs that correspond to memory stored in the paging area (and thus not in RAM) as invalid. If an application tries to use one of the pages that has been paged out to disk, the application will incur a page fault, and the page must be paged in: the kernel trap handler will allocate a page of physical RAM, read the page from disk into the RAM, and modify the relevant PTE to point to the RAM.

What happens if a page needs to be paged in, but there is no free physical RAM? In that case, the kernel must first free a physical page by paging it out or evicting it to the paging area on disk, and marking the PTEs referring to that physical page as invalid.