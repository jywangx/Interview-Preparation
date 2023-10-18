# Linux中Pipe与Splice系统调用原理

> [An In-Depth Look at Pipe and Splice implementation in Linux kernel](https://blogs.oracle.com/linux/post/pipe-and-splice)

## Pipe

pipe（管道）是一种在Unix和Linux系统中用于进程间通信的机制，它允许一个进程将数据传递给另一个进程。管道通常用于创建简单的数据传输通道，其中一个进程充当数据的生产者，而另一个进程充当数据的消费者。

pipe的内部使用维护了一个环形缓冲区，环形缓冲区的元素由`pipe_buffer`定义：

```c
struct pipe_buffer {
    struct page *page; //指向存放pipe buffer data的实际页面的指针
    unsigned int offset, len; //页面中可操作的部分
    const struct pipe_buf_operations *ops;
    unsigned int flags;
    unsigned long private;
};
```

pipe本身由`pipe_inode_info`描述：

```c
struct pipe_inode_info {
    unsigned int head;
    unsigned int tail;
    unsigned int ring_size;
    struct pipe_buffer *bufs; //环形缓冲区
    ............
};
```

其结构示意图如下：

![image-20231017160410635](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/image-20231017160410635.png)

### Pipe Write

用户通过`write`向pipe写入时，Linux内核会调用`pipe_write`，向head的`pipe_buffer`中写入数据。内部步骤：

1. 选择当前head指针，并通过`was_empty`记录当前管道是否为空，之后用来通知管道的reader；
2. 分配一个页面，通过`pipe_full`检测管道是否已满，若未满head移动到下一位；
3. 将分配好的page插入到head指向的缓冲区中，并将用户缓冲区的数据拷贝进分配好的page；
4. 重复上述过程直到所有数据都被写入。如果没有空闲的`pipe_buffer`会等待直到有空闲的可用；如果写入之前`was_empty`记录到管道为空，就会通知正在等待读取的进程从pipe中读取。

![image-20231017194454469](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/image-20231017194454469.png)

上述步骤对应的部分源码：

```c
// 步骤1
head = pipe->head;
was_empty = pipe_empty(head, pipe->tail);

for (;;) {
    head = pipe->head;
    if (!pipe_full(head, pipe->tail, pipe->max_usage)) {
        unsigned int mask = pipe->ring_size - 1;
        struct pipe_buffer *buf = &pipe->bufs[head & mask];
        struct page *page = NULL;
        int copied;
		
        // 步骤2
        if (!page) {
            page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT);
            if (unlikely(!page)) {
                ret = ret ? : -ENOMEM;
                break;
            }
        }

        /* Allocate a slot in the ring in advance and attach an
         * empty buffer.  If we fault or otherwise fail to use
         * it, either the reader will consume it or it'll still
         * be there for the next write.
         */
        spin_lock_irq(&pipe->rd_wait.lock);

        head = pipe->head;
        if (pipe_full(head, pipe->tail, pipe->max_usage)) {
            spin_unlock_irq(&pipe->rd_wait.lock);
            continue;
        }

        pipe->head = head + 1;
        spin_unlock_irq(&pipe->rd_wait.lock);
        
        // 步骤3
        /* Insert it into the buffer array */
        buf = &pipe->bufs[head & mask];
        buf->page = page;
        buf->ops = &anon_pipe_buf_ops;
        buf->offset = 0;
        buf->len = 0;
        if (is_packetized(filp))
            buf->flags = PIPE_BUF_FLAG_PACKET;
        else
            buf->flags = PIPE_BUF_FLAG_CAN_MERGE;

        copied = copy_page_from_iter(page, 0, PAGE_SIZE, from);
        if (unlikely(copied < PAGE_SIZE && iov_iter_count(from))) {
            if (!ret)
                ret = -EFAULT;
            break;
        }

        ret += copied;
        buf->offset = 0;
        buf->len = copied;

        if (!iov_iter_count(from))
            break;
    }
    
}

// 通知reader
if (was_empty)
           wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
wait_event_interruptible_exclusive(pipe->wr_wait, pipe_writable(pipe));
```

### Pipe Read

从管道中读取数据是通过 read 系统调用实现的。它在内部也是调用 pipe_read 函数。步骤：

1. 相似的，开始操作之前先检验管道是否已经满了，设置was_full；
2. 检验管道状态与剩余数据长度：
   - 数据读取完毕：退出循环；
   - 管道为空且没有writer：退出循环；
   - 不为空且数据未读取完毕：读取数据，移动tail；
   - 读取操作还会等待管道的rd_wait事件，因为写入者可能会向管道写入一些数据。

![image-20231017210147547](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/image-20231017210147547.png)

相关源码：

```c
was_full = pipe_full(pipe->head, pipe->tail, pipe->max_usage);
for (;;) {
    /* Read ->head with a barrier vs post_one_notification() */
    unsigned int head = smp_load_acquire(&pipe->head);
    unsigned int tail = pipe->tail;
    unsigned int mask = pipe->ring_size - 1;
    if (!pipe_empty(head, tail)) {
        // 拷贝数据
        struct pipe_buffer *buf = &pipe->bufs[tail & mask];
        size_t chars = buf->len;
        size_t written;
        int error;

        error = pipe_buf_confirm(pipe, buf);
        if (error) {
            if (!ret)
                ret = error;
            break;
        }

        written = copy_page_to_iter(buf->page, buf->offset, chars, to);
        if (unlikely(written < chars)) {
            if (!ret)
                ret = -EFAULT;
            break;
        }
        ret += chars;
        buf->offset += chars;
        buf->len -= chars;
        
        // 当前pipe buffer的数据已被读完，移动tail到下一个位置
        if (!buf->len) {
            spin_lock_irq(&pipe->rd_wait.lock);
            tail++;
            pipe->tail = tail;
            spin_unlock_irq(&pipe->rd_wait.lock);
        }
        total_len -= chars;
        
        // 数据全部读取完毕
        if (!total_len)
            break;    /* common path: read succeeded */
        
        // 管道为空但数据没读完则继续执行
        if (!pipe_empty(head, tail))    /* More to do? */
            continue;
    }
    // 
    if (!pipe->writers)
        break;
    if (ret)
        break;
    if (filp->f_flags & O_NONBLOCK) {
        ret = -EAGAIN;
        break;
    }
    
}
if (pipe_empty(pipe->head, pipe->tail))
    wake_next_reader = false;
__pipe_unlock(pipe);

// 通知writer进程
if (was_full)
    wake_up_interruptible_sync_poll(&pipe->wr_wait, EPOLLOUT | EPOLLWRNORM);
// 唤醒其他等待的reader
if (wake_next_reader)
    wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN | EPOLLRDNORM);
kill_fasync(&pipe->fasync_writers, SIGIO, POLL_OUT);
if (ret > 0)
    file_accessed(filp);
// 返回读取的字节数
return ret;
```

## Splice

splice提供了一种使用管道读取或写入文件描述符的有效方法，它在两个文件描述符之间移动数据，无需在内核地址空间和用户地址空间之间进行复制，两个文件描述符中至少有一个需要是管道。也就是说它支持三种传输形式：

1. pipe to pipe
2. pipe to file
3. file to pipe



### pipe to pipe

管道间的splice是最简单的一种形式，无需数据拷贝，如下图，Source pipe和Destination pipe初始情况如下：

![image-20231017220340158](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/image-20231017220340158.png)

pipe间的splice直接把source pipe的page信息拷贝到destination pipe，不进行真正的数据拷贝：

![image-20231017220853686](https://redblog.oss-cn-chengdu.aliyuncs.com/picgo/image-20231017220853686.png)

### file to pipe&pipe to file

文件到管道的splice允许文件与管道共享page cache data buffer，避免了管道从文件读取数据时的数据复制；反之管道到文件的splice遍历所有包含数据的管道缓冲区槽，并调用 VFS（虚拟文件系统）例程`vfs_iter_write`将内容写入输出文件来实现。
