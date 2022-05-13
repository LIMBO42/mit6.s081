#### 操作系统结构：

![image-20220421151908648](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220421151908648.png)

分层：用户空间，kernel，硬件

kernel管理硬件资源，进程用户空间以及文件系统（processes，mem allocation， access control）

用户空间通过system call获得kernel提供的服务。

qemu：硬件模拟器

question：进程先open获得文件描述符，然后fork，子进程能访问文件描述符吗？能

- xv6
- RISC-V
- QEMU

xv6运行在QEMU模拟的RISC-V微处理器上



#### 系统调用

**1.read()：**

```c
int n = read(0,buf,sizeof(buf));
```

第一个参数为文件描述符，0标准输入，1标准输出；

第二个参数为指针指向一段内存地址；

第三个参数为read预计读取的字节数量；

返回值为真正读取到的字节数量。

注意，如果`read(0,buf,sizeof(buf)+1)`，则读入的字节会冲掉栈上的某些内容。

**write()：**

```C
write(1,buf,n);
```

**2.shell：**注意<的重定向。

```bash
grep name < output.txt
```

**3.fork()：**

状态机的复制，除了返回的pid不一样，其他全都一样

**4.exec()：**

```bash
char *argv[]={"echo", "this", "is", "echo", 0};
exec(filepath, argv);
```

参数：第一个参数为可执行文件的路径，第二个参数为指令；

**注意**：argv[0]为指令的名字，且需要0/null来指定数组结尾。

exec()重置状态机，加载可执行文件的指令进内存运行；但是会**保留打开的文件描述符**，另外exec()调用不会返回（除了找不到可执行文件）。

![image-20220421163347946](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220421163347946.png)

fork/exec程序：注意到exec执行成功不会执行15，16行的内容；另外父进程用wait()函数来等待子进程，status为子进程的返回值。

**思考：**fork复制状态机（消耗资源），但是exec立马又重置状态机

解决：copy-on-write，利用指针指向某些页面形成复用，只复制exec需要的内存



**5.wait()**：

父进程等待，直到有子进程退出。多个子进程则需要多个wait



**6.redirect**

![image-20220421183656983](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220421183656983.png)

这段代码实现了将输出"this is redirected"重定向到output.txt的文件的功能。

子进程先关闭了1，再open output.txt，因为open返回当前未被分配的最小的文件描述符，因此标准输出1现在是重定向为了output.txt，然后exec继承了文件描述符表。