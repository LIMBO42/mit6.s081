### lock

#### 为什么需要锁

多cpu的并行环境下，需要锁来保证共享数据的一致性

![image-20220510090216276](https://typora-1306385380.cos.ap-nanjing.myqcloud.com/img/image-20220510090216276.png)

acquire和release之间的代码称之为critical section，以原子的方式执行共享数据的更新。