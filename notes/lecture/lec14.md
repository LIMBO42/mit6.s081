inode使用编号来进行区分，inode必须有一个link count来跟踪指向这个inode的文件名的数量。一个文件（inode）只能在link count为0的时候被删除。还有一个openfd count，也就是当前打开了文件的文件描述符计数。一个文件只能在这两个计数器都为0的时候才能被删除。文件描述符还需要维护offset。

![image-20220512100202421](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512100202421.png)

![image-20220512100236048](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512100236048.png)

通常来说，bitmap block，inode blocks和log blocks被统称为metadata block。它们虽然不存储实际的数据，但是它们存储了能帮助文件系统完成工作的元数据。

![image-20220512101019358](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512101019358.png)

![image-20220512101218851](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512101218851.png)

目录：inode以及inode对应的名字

如何查找目录：从目录开始去看entry。



#### xv6文件系统

启动XV6的过程中，调用了makefs指令，来创建一个文件系统。

![image-20220512101830120](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512101830120.png)

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512102746653.png" alt="image-20220512102746653" style="zoom:50%;" />

<img src="https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220512102752414.png" alt="image-20220512102752414" style="zoom: 50%;" />

如图：前两个写block33，第一个是为了标记inode将要被使用。在XV6中，使用inode中的type字段来标识inode是否空闲，这个字段同时也会用来表示inode是一个文件还是一个目录。所以这里将inode的type从空闲改成了文件，并写入磁盘表示这个inode已经被使用了。第二个write 33就是实际的写入inode的内容。inode的内容会包含linkcount为1以及其他内容。

写46是因为46存储的是根目录，我们向根目录增加了一个新的entry，其中包含了文件名x，以及我们刚刚分配的inode编号。

写32是因为32对应根目录的inode，根目录的大小发生了变化，所以修改inode32中的size字段。

write45是因为寻找没被使用的datablock，两次写595是将hi写入block 595。最后的write 33是更新文件x对应的inode中的size字段，因为现在文件x中有了两个字符。