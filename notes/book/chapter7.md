#### 线程切换

The function swtch performs the saves and restores for a kernel thread switch. swtch doesn’t directly know about threads; it just saves and restores sets of 32 RISC-V registers, called contexts. When it is time for a process to give up the CPU, the process’s kernel thread calls swtch to save its own context and return to the scheduler context. Each context is contained in a struct context (kernel/proc.h:2), itself contained in a process’s struct proc or a CPU’s struct cpu. Swtch takes two arguments: struct context *old and struct context *new. It saves the current registers in old, loads registers from new, and returns.

swtch只用保存callee-saved register，因为swtch就像函数一样被调用，其他caller-saved register会被保存在stack里，而swtch保存的寄存器sp就指明了栈的位置。

swtch还保存了ra寄存器，ra保存函数返回的地址。When swtch returns, it returns to the instructions pointed to by the restored ra register, that is, the instruction from which the new thread previously called swtch.

这个new thread之前调用的指令地址在哪里？

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220510162323116.png" alt="image-20220510162323116" style="zoom:50%;" />

因此会返回scheduler函数（这个new thread事实上是内核的scheduler线程），该函数会去寻找合适的用户线程调度。

![image-20220510163400982](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220510163400982.png)

 Procedures that intentionally transfer control to each other via thread switch are sometimes referred to as coroutines;



allocproc sets the context ra register of a new process to forkret (kernel/proc.c:508), so that its first swtch “returns” to the start of that function. A fork child's very first scheduling by scheduler()  will swtch to forkret.

![image-20220510182640644](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220510182640644.png)

创建线程之后被调度执行会去执行forkret，由于该进程不是通过switch被调度上来的，所以需要之后执行usertrapret返回用户态。



#### PV操作

The xv6 kernel uses a mechanism called sleep and wakeup in these situations (and many others). Sleep allows a kernel thread to wait for a specific event; another thread can call wakeup to indicate that threads waiting for an event should resume. Sleep and wakeup are often called sequence coordination or conditional synchronization mechanisms

xv6用的sleep和wakeup + 数量相当于其他的条件同步机制PV

<img src="C:\Users\LIMBO\AppData\Roaming\Typora\typora-user-images\image-20220511090401076.png" alt="image-20220511090401076" style="zoom:50%;" />

队列，唤醒

412行的acquire是为了保护413行的count的原子性；而414行新加的是保证lock锁带进sleep函数，这样在进程sleep之后可以释放掉这个锁，在V的时候才能获得锁，才能正确唤醒。（否则，V获得不到锁，这样就会死锁）

<img src="C:\Users\LIMBO\AppData\Roaming\Typora\typora-user-images\image-20220511154920358.png" alt="image-20220511154920358" style="zoom:50%;" />

关于锁，获得和释放，获得p->lock是为了进程切换的完整性，而释放lk是为了wakeup能唤醒。

为什么不在sleep和while之间释放锁？会导致lost wake up问题。



The basic idea is to have sleep mark the current process as SLEEPING and then call sched to release the CPU; wakeup looks for a process sleeping on the given wait channel and marks it as RUNNABLE.

![image-20220511092113825](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220511092113825.png)

sleep和wake之间的锁的关系，需要保证更改进程装态为sleeping的时候的操作，wakeup没办法唤醒该进程

不管是sleep还是yeild，在获取了当前进程的锁之后，完成swtch之后，新运行的进程都会是scheduler，返回上次swtch的位置，然后释放锁。

![image-20220511100015358](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220511100015358.png)

#### pipe机制

A more complex example that uses sleep and wakeup to synchronize producers and consumers is xv6’s implementation of pipes. We saw the interface for pipes in Chapter 1: bytes written to one end of a pipe are copied to an in-kernel buffer and then can be read from the other end of the pipe.

pipe是循环队列，由data和lock构成

Let’s suppose that calls to piperead and pipewrite happen simultaneously on two different CPUs. Pipewrite (kernel/pipe.c:77) begins by acquiring the pipe’s lock, which protects the counts, the data, and their associated invariants. Piperead (kernel/pipe.c:106) then tries to acquire the lock too, but cannot. It spins in acquire (kernel/spinlock.c:22) waiting for the lock. While piperead waits, pipewrite loops over the bytes being written (addr[0..n-1]), adding each to the pipe in turn (kernel/pipe.c:95). During this loop, it could happen that the buffer fills (kernel/pipe.c:88). In this case, pipewrite calls wakeup to alert any sleeping readers to the fact that there is data waiting in the buffer and then sleeps on &pi->nwrite to wait for a reader to take some bytes out of the buffer. Sleep releases pi->lock as part of putting pipewrite’s process to sleep.

Now that pi->lock is available, piperead manages to acquire it and enters its critical section: it finds that pi->nread != pi->nwrite (kernel/pipe.c:113) (pipewrite went to sleep because pi->nwrite == pi->nread+PIPESIZE (kernel/pipe.c:88)), so it falls through to the for loop, copies data out of the pipe (kernel/pipe.c:120), and increments nread by the number of bytes copied. That many bytes are now available for writing, so piperead calls wakeup (kernel/pipe.c:127) to wake any sleeping writers before it returns. Wakeup finds a process sleeping on &pi->nwrite, the process that was running pipewrite but stopped when the buffer filled. It marks that process as RUNNABLE.

write n 个字符满了的话sleep，并唤醒read；read空的话唤醒write这样。



#### 进程的wait, kill以及exit

在子进程exit和父进程在wait发现子进程exit之间的子进程的状态为zombie，直到父进程发现子进程exit，才会将子进程状态改为unused，copies the child’s exit status, and returns the child’s process ID to the parent。但是如果子进程没完成，父进程就死掉了，子进程就被转给init进程。

![image-20220511105201260](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220511105201260.png)

kill设置p->killed为1，同时唤醒sleeping的进程，这样在usertrap里面调用exit就能够退出该进程（sleeping的进程被唤醒最后一定会进入usertrap）。

