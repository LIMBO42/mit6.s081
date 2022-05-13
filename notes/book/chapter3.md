## page table

page tables 给进程提供了一种抽象，好像他们有独立的内存地址空间；把各个进程内存地址空间隔离开的同时，映射到一个物理地址空间

xv6 performs a few tricks: mapping the same memory (a trampoline page) in several address spaces, and guarding kernel and user stacks with an unmapped page.



#### 硬件

1.RSCV使用39位作为虚拟地址，一页有4KB，用12位，因此有27位用作page table entries(PTE)，每个PTE包含44位的物理页号(physical page number, PPN)，这样物理地址是56位。低12位用来指示地址在那一页中的位置，直接复制即可。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220429091500591.png" alt="image-20220429091500591" style="zoom:67%;" />

2.多级地址翻译：

每一级使用9位作为索引，$2^9 = 512$，因此每一级索引512个PTE，如果要翻译的地址查不到，会引起   ***page fault exception***   。

***多级索引的好处：***比如进程只使用了一级 page  directory 的一个目录项，那么不使用的那些目录项的二级三级 page directory就不用分配空间。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220429092732135.png" alt="image-20220429092732135" style="zoom:67%;" />

PTE_V indicates whether the PTE is present: if it is not set, a reference to the page causes an exception (i.e. is not allowed). PTE_R controls whether instructions are allowed to read to the page. PTE_W controls whether instructions are allowed to write to the page. PTE_X controls whether the CPU may interpret the content of the page as instructions and execute them. PTE_U controls whether instructions in user mode are allowed to access the page; if PTE_U is not set, the PTE can be used only in supervisor mode. Figure 3.2 shows how it all works. The flags and all other page hardware-related structures are defined in (kernel/riscv.h)

3.SATP：保存 the root page-table page的物理地址



### kernel address space

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220429093228654.png" alt="image-20220429093228654" style="zoom:67%;" />

Xv6 maintains **one page table per process**, describing each process’s user address space, plus **a single page table that describes the kernel’s address space**.

kernel设置内存地址空间，去访问物理地址和硬件资源。

QEMU simulates a computer that includes RAM (physical memory) starting at physical address 0x80000000 and continuing through at least 0x86400000, which xv6 calls PHYSTOP.

0x80000000以下作为 **device interface memory-mapped control register** . The kernel can interact with the devices by reading/writing these special physical addresses

RAM和 memory mapped device register 使用的是 direct napping。虚拟地址和物理地址映射是相同的。$0-0x86400000$

 <img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220430083538932.png" alt="image-20220430083538932" style="zoom:67%;" />

- The trampoline page：映射了两次，一次是 top of the virtual address，另外一次是直接映射。
- kernel stack page：因为每个进程都有一个kernel stack，一个user stack，每个 kernel stack page 都接上一个 guard page，kernel page映射了两次，guard 不映射。



### Code：creating an address space

pagetable_t 指针，指向页目录的起始位置，512个页目录项。

walk：根据虚拟地址去找PTE

mappages：根据虚拟地址去创建PTE

kvm开头的函数关于 kernel page table，uvm是关于 user page table。

copyin：从 user copy 到 kernel；copyout：从 kernel copy 到 user。

在启动OS的时候，main函数调用kvminit，然后再调用kvmmake，创建

page table，然后kvmmap创建PTE，映射 kernel’s instructions and data, physical memory up to PHYSTOP and memory ranges which are actually devices。

proc_mapstacks会给每个进程分配kernel stack，同样是利用kvmap创建PTE。

kvmmap (kernel/vm.c:127) calls mappages (kernel/vm.c:138), which installs mappings into a page table for a range of virtual addresses to a corresponding range of physical addresses. 

For each virtual address to be mapped, mappages calls walk to find the address of the PTE for that address. It then initializes the PTE to hold the relevant physical page number, the desired permissions (PTE_W, PTE_X, and/or PTE_R), and PTE_V to mark the PTE as valid (kernel/vm.c:153).

相当于是给出了物理地址，mappages函数会计算出对应的PTE，填入table即可。

walk (kernel/vm.c:81) mimics the RISC-V paging hardware as it looks up the PTE for a virtual address (see Figure 3.2). walk descends the 3-level page table 9 bits at the time. It uses each level’s 9 bits of virtual address to find the PTE of either the next-level page table or the final page (kernel/vm.c:87). If the PTE isn’t valid, then the required page hasn’t yet been allocated; if the alloc argument is set, walk allocates a new page-table page and puts its physical address in the PTE. It returns the address of the PTE in the lowest layer in the tree (kernel/vm.c:97).

The above code depends on physical memory **being direct-mapped into the kernel virtual address space.** For example, as walk descends levels of the page table, it pulls the (physical) address of the next-level-down page table from a PTE (kernel/vm.c:89), and then uses that address as a virtual address to fetch the PTE at the next level down (kernel/vm.c:87).

main calls kvminithart (kernel/vm.c:62) to install the kernel page table. It writes the physical address of the root page-table page into the register satp. After this the CPU will translate addresses using the kernel page table. Since the kernel uses an identity mapping, the now virtual address of the next instruction will map to the right physical memory address.

调用kvminithart将基地址装入SATP，然后开启了分页，地址变为虚拟地址，但由于映射是identity，因此下一条指令可以直接看做虚拟地址。

利用TLB作为cache存储虚拟地址向物理地址的映射关系，切换page table的时候，TLB就invalid了。

xv6 uses the physical memory between the end of the kernel and PHYSTOP for run-time allocation.

### Code: Physical memory allocator

每个进程有自己的page table，当切换进程的时候，就切换page table。

When a process asks xv6 for more user memory, xv6 first uses kalloc to allocate physical pages. It then adds PTEs to the process’s page table that point to the new physical pages.

使用kalloc分配物理页面，添加PTEs。

因为每个进程有自己的页表，因此可以将相同的虚拟地址翻译到不同的物理地址，提供了进程间的隔离。另外给进程提供了地址好像是连续的这样一种错觉。

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220430095907137.png" alt="image-20220430095907137" style="zoom:67%;" />

The stack is a single page, and is shown with the initial contents as created by exec. Strings containing the command-line arguments, as well as an array of pointers to them, are at the very top of the stack. Just under that are values that allow a program to start at main as if the function main(argc, argv) had just been called.

stack一开始就包含了main的参数，然后栈向下增长。

To detect a user stack overflowing the allocated stack memory, xv6 places an **inaccessible guard page** right below the stack by clearing the PTE_U flag

Sbrk is the system call for a process to shrink or grow its memory. The system call is implemented by the function growproc (kernel/proc.c:253). growproc calls uvmalloc or uvmdealloc, depending on whether n is postive or negative. uvmalloc (kernel/vm.c:221) allocates physical memory with kalloc, and adds PTEs to the user page table with mappages. uvmdealloc calls uvmunmap (kernel/vm.c:166), which uses walk to find PTEs and kfree to free the physical memory they refer to.



### sbrk

sbrk是进程用来grow or shrink 内存空间的系统调用。

如何增加？kalloc分配物理地址空间，然后添加PTE。



### exec

creates the user part of an address space

根据可执行文件创建用户的地址空间，elf格式的文件，然后有program section header，用来描述需要被加载进内存的信息。

exec会分配page table with no mapping，然后根据elf的segment分配空间，将程序加载进内存。接着exec会分配用户栈，将argc，argv加载进栈。

另外，如果在加载内存镜像的时候出现问题是需要恢复成原来的镜像的，所以只有在所有的加载完成之后才能free old one。

exec的行为就是根据文件中标注的地址信息，将bytes from elf file 加载进内存。如果用户的内存地址空间和kernel的内存地址空间不分离的话，由于将bytes加载到文件指定的地址，有可能会有风险。

现在的xv6 kernel address和user address是分开的，kernel 有自己的page table。

所以其实内存地址空间是通过page table的映射关系来体现的。