## Linux下零拷贝机制

> [浅谈 Linux下的零拷贝机制](https://www.jianshu.com/p/e76e3580e356)

### 零拷贝实现

1. DMA
2. sendfile
3. splice
4. mmap

#### DMA

外设通过DMA引擎将IO数据直接传入RAM，无需CPU参与。

#### sendfile

`sendfile`是一个系统调用，用于在文件之间进行数据传输，而无需通过用户空间进行中间缓冲。步骤：

1. 发起sendfile系统调用，从**用户态切换到内核态**，通过DMA将文件位于磁盘的数据拷贝到内核数据缓冲区，再由**CPU**将数据拷贝到socket的数据缓冲区；
2. 系统调用返回，**切换回用户态**，通过**DMA**将数据传给协议栈处理。

共涉及2次上下文切换和3次数据传输，避免了数据拷贝到用户态数据缓冲区的过程。

![img](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/4235178-66c23adafbfbd47f.jpeg)

消除剩余的一次CPU拷贝：使用基于scatter/gather的DMA来从内核数据缓冲区直接拷贝到protocol engine的空间。

![img](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/4235178-df9323d3ae59b8f8.jpeg)

#### splice

利用splice进行socket间的零拷贝数据传输需要借助一个pipe作为中介：

```c
splice(socket1_fd, pipe_fd   );
splice(pipe_fd   , socket2_fd);
```

#### mmap

mmap(内存映射)是一个比sendfile昂贵但优于传统I/O的方法。

![img](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/4235178-2700bead4cf14739.jpeg)

1. 发起mmap系统调用，用户空间**切换**到内核空间，**DMA**将磁盘数据拷贝到kernel buffer；
2. mmap返回，内核空间**切换**用户空间，用户空间和内核空间共享缓冲区；
3. 发起write系统调用，用户空间切换到内核空间，**CPU**将数据拷贝到socket的缓冲区；
4. write返回，内核空间**切换**用户空间，**DMA**将数据写入protocol engine。
