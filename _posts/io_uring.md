---
title: io_uring
date: 2024-2-05
tags:
- read
---



# io_uring

在linux 5.1 版本之后，Linux内核提供了异步IO的框架支持，提供了三个系统调用 ` io_uring_enter  io_uring_register  io_uring_setup` 在liburing.h头文件中对此系统调用进行了封装，如：

```c
int io_uring_enter(int ring_fd, unsigned int to_submit,
                   unsigned int min_complete, unsigned int flags)
{
    return (int) syscall(__NR_io_uring_enter, ring_fd, to_submit, min_complete,
                         flags, NULL, 0);
}
```

其中系统调用号` __NR_io_uring_enter ` 在` unistd_64.h `中被定义：` #define __NR_io_uring_enter 426 `

## 性能：

由于调用系统调用时，会从用户态切换到内核态，从而进行上下文切换，而上下文切换会消耗一定的 CPU 时间。` io_uring ` 为了减少或者摒弃系统调用，采用了用户态与内核态共享内存的方式来通信(用户态对共享内存进行读写操作是不需要使用系统调用的，所以不会发生上下文切换的情况)。



## 初始化

```c
int io_uring_setup(u32 entries, struct io_uring_params *p);
```

用户通过调用 `io_uring_setup `初始化一个新的 `io_uring` 上下文。该函数返回一个 file descriptor，并将 `io_uring` 支持的功能、以及各个数据结构在 `fd` 中的偏移量存入 `params`。用户根据偏移量将 `fd` 映射到内存 (mmap) 后即可获得一块内核用户共享的内存区域。这块内存区域中，有 `io_uring` 的上下文信息：提交队列信息 (`SQ_RING`) 和完成队列信息 (`CQ_RING`)；还有一块专门用来存放提交队列元素的区域 (SQEs)。`SQ_RING` 中只存储 SQE 在 SQEs 区域中的序号，`CQ_RING` 存储完整的任务完成数据。

![](/img/v2-ad01522fd88442e9164001926b3d839c_r.png)

`io_uring` 在创建时有两个选项(flag)，对应着 `io_uring` 处理任务的不同方式：

- 开启 `IORING_SETUP_IOPOLL` 后，`io_uring` 会使用轮询的方式执行所有的操作。
- 开启 `IORING_SETUP_SQPOLL` 后，`io_uring` 会创建一个内核线程专门用来收割用户提交的任务。
- 都不开启，通过 `io_uring_enter` 提交任务，收割任务无需 syscall。

`io_uring_setup` 设计的巧妙之处在于，内核通过一块和用户共享的内存区域进行消息的传递。在创建上下文后，任务提交、任务收割等操作都通过这块共享的内存区域进行，在 `IO_SQPOLL` 模式下，可以完全绕过 Linux 的 syscall 机制完成需要内核介入的操作（比如读写文件），大大减少了 syscall 切换上下文、刷 TLB 的开销。



## 任务的定义

` io_uring ` 定义的异步 io 请求，对应宏定义

```c
#define IORING_OP_NOP		0
#define IORING_OP_READV		1
#define IORING_OP_WRITEV	2
#define IORING_OP_FSYNC		3
```

内核中定义了` io_op_defs `数组用与描述对应的异步 io 请求所需要的条件

```c
static const struct io_op_def io_op_defs[] = {
	[IORING_OP_NOP] = {},
	[IORING_OP_READV] = {
		.needs_file		= 1,				//表示该操作需要文件描述符。
		.unbound_nonreg_file	= 1,		 //表示该操作需要非正则文件
		.pollin			= 1,				//表示该操作需要poll in。
		.buffer_select		= 1,			//表示该操作需要buffer_select
		.needs_async_setup	= 1,			//表示该操作需要异步设置。
		.plug			= 1,			   //表示该操作需要plug。	 
		.async_size		= sizeof(struct io_async_rw),	//表示该操作的异步结构体大小
	},	
	[IORING_OP_WRITEV] = {
		.needs_file		= 1,
		.hash_reg_file		= 1,
		.unbound_nonreg_file	= 1,
		.pollout		= 1,
		.needs_async_setup	= 1,
		.plug			= 1,
		.async_size		= sizeof(struct io_async_rw),
	},
    ....
}
```

`io_uring` 中几乎每个操作都有对应的准备和执行函数。比如 `read` 操作就对应 `io_read_prep` 和 `io_read` 函数。除了同步操作，内核还支持异步调用的操作，对于这些操作，` io_uring `中还会有一个对应的异步准备函数以 `_async` 结尾

```c
static int io_sendmsg_prep_async(struct io_kiocb *req)
static inline int io_rw_prep_async(struct io_kiocb *req, int rw);
```



## 任务的创建

**用户将需要进行的操作写入 `io_uring` 的 SQ 中。在 CQ 中，用户可以收割任务的完成情况。**

sqe的结构：

```c
struct io_uring_sqe {
	__u8	opcode;		/* type of operation for this sqe */
	__u8	flags;		/* IOSQE_ flags */
	__u16	ioprio;		/* ioprio for the request */
	__s32	fd;		/* file descriptor to do IO on */
	__u64	off;		/* offset into file */
	__u64	addr;		/* pointer to buffer or iovecs */
	__u32	len;		/* buffer size or number of iovecs */
	union {
		__kernel_rwf_t	rw_flags;
		__u32		fsync_flags;
		__u16		poll_events;
		__u32		sync_range_flags;
		__u32		msg_flags;
		__u32		timeout_flags;
	};
	__u64	user_data;	/* data to be passed back at completion time */
	union {
		__u16	buf_index;	/* index into fixed buffers, if used */
		__u64	__pad2[3];
	};
};
```

如要进行` readv ` 操作：
```c
    sqe->fd = filefd;						//需要操作的文件描述符
    sqe->flags = 0;							
    sqe->opcode = IORING_OP_READV;			 //readv对应option_code
    sqe->addr = &iovecs;	  				//存放的起始地址
    sqe->len = blocks;						//对应 iovec 数组的长度
    sqe->off = 0;							//从文件偏移位置为0处开始
```

通常来说，使用 `io_uring` 的程序都需要用到 64 位的 `user_data` 来唯一标识一个操作。`user_data` 是 SQE 的一部分。`io_uring` 执行完某个操作后，会将这个操作的 `user_data` 和操作的返回值一起写入 CQ 中。一般携带指向堆内存的指针

```c
struct io_uring_cqe {
	__u64	user_data;	/* sqe->data submission passed back */
	__s32	res;		/* result code for this event */
	__u32	flags;		// 未使用
};
```

 `io_ uring` 是一个异步接口，`errno` 将不用于传回错误信息。与此对应，`res` 将保存在成功的情况下等效的系统调用将要返回的内容，而在出错的情况下 `res` 将包含`-errno`。例如，如果正常读取系统调用返回`-1`并将`errno`设置为`EINVAL`，则`res`将包含`-EINVAL`。



## 任务的提交与收割

`io_uring` 通过环形队列和用户交互。

![](/img/v2-f688ede3a66c848fb4e3333767dfd9cc_r.png)

我们的先以用户提交任务为例，介绍 `io_uring` 的内核用户交互方式。用户提交任务的过程如下：

- 将 SQE 写入 SQEs 区域，而后将 SQE 编号写入 SQ。（对应图中绿色第一步）
- 更新用户态记录的队头。（对应图中绿色第二步）
- 如果有多个任务需要同时提交，用户不断重复上面的过程。
- 将最终的队头编号写入与内核共享的 `io_uring` 上下文。（对应图中绿色第三步）

接下来我们简要介绍内核获取任务、内核完成任务、用户收割任务的过程。

- 内核态获取任务的方式是，从队尾读取 SQE，并更新 `io_uring` 上下文的 SQ tail。

![](/img/v2-670198a5e28380ee33809eec41d39e04_r.png)

- 内核态完成任务：往 CQ 中写入 CQE，更新上下文 CQ head。
- 用户态收割任务：从 CQ 中读取 CQE，更新上下文 CQ tail。



`io_uring` 在创建时有两个选项，对应着 `io_uring` 处理任务的不同方式：

- 开启 `IORING_SETUP_IOPOLL` 后，`io_uring` 会使用轮询的方式执行所有的操作。
- 开启 `IORING_SETUP_SQPOLL` 后，`io_uring` 会创建一个内核线程专门用来收割用户提交的任务。

这些选项的设定会影响之后用户与 `io_uring` 交互的方式：

- 都不开启，通过 `io_uring_enter` 提交任务，收割任务无需 syscall。
- 只开启 `IORING_SETUP_IOPOLL`，通过 `io_uring_enter` 提交任务和收割任务。
- 开启 `IORING_SETUP_SQPOLL`，无需任何 syscall 即可提交、收割任务。内核线程在一段时间无操作后会休眠，可以通过 `io_uring_enter` 唤醒。



## liburing

使用流程：

1. 使用io_uring_queue_init，完成io_uring相关结构的初始化。在这个函数的实现中，会调用多个mmap来初始化一些内存。

2. 初始化完成之后，为了提交IO请求，需要获取里面queue的一个项，使用io_uring_get_sqe。

3. 获取到了空闲项之后，使用io_uring_prep_readv、io_uring_prep_writev初始化读、写请求。和前文所提preadv、pwritev的思想差不多，这里直接以不同的操作码委托io_uring_prep_rw，io_uring_prep_rw只是简单地初始化io_uring_sqe。

4. 准备完成之后，使用io_uring_submit提交请求。

5. 提交了IO请求时，可以通过非阻塞式函数io_uring_peek_cqe、阻塞式函数io_uring_wait_cqe获取请求完成的情况。默认情况下，完成的IO请求还会存在内部的队列中，需要通过io_uring_cqe_seen表标记完成操作。

6. 使用完成之后要通过io_uring_queue_exit来完成资源清理的工作。

### link operation

在io_uring中完成的任务并不是按照提交顺序返回的，有时我们需要按顺序的完成一组任务，这需要设置` sqe `对应的flag，为` flag `添加 ` IOSQE_IO_LINK `

```c
    io_uring_prep_write(sqe, fd, STR, strlen(STR), 0 );
    sqe->flags |= IOSQE_IO_LINK;						//添加link flag

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_read(sqe, fd, buff, strlen(STR),0);
    sqe->flags |= IOSQE_IO_LINK;						//添加link flag

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_close(sqe, fd);
    io_uring_submit(ring);
```

` IOSQE_IO_LINK `使得本` sqe `与下一提交的` sqe `相关联，即两个任务之间有了先后顺序，如上代码就保证了，先读后写最后关闭

倘若我们操作的文件没有对应的权限，如没有写权限，文件以 O_WRONLY 打开，那么read操作将失败，这将导致后续link的操作全部失败

当涉及链接操作时，一个操作的失败将导致所有后续链接操作失败，并出现 errno“Operation cancelled”

### regster

注册文件或用户缓冲区允许内核长期引用内部数据结构或创建应用程序内存的长期映射，从而大大减少每个I/O的开销。

应用程序可以增加或减少已注册缓冲区的大小或数量，方法是首先取消注册现有缓冲区，然后使用新缓冲区发出对io_uring_register（）的新调用。注册缓冲区将等待环空闲。如果应用程序当前有正在处理的请求，注册将等待这些请求完成后再继续。

```c
int io_uring_register_buffers(struct io_uring *ring, const struct iovec *iovecs, unsigned nr_iovecs);
void io_uring_prep_write_fixed(struct io_uring_sqe *sqe,
                               int fd,
                               const void *buf,
                               unsigned nbytes,
                               __u64 offset,
                               int buf_index);
 void io_uring_prep_read_fixed(struct io_uring_sqe *sqe,
                               int fd,
                               void *buf,
                               unsigned nbytes,
                               __u64 offset,
                               int buf_index);
```

在register后，对映射后的内存进行read / write 操作时，避免一次数据copy，register可以理解为将iovec mmap 到内核中，这样在进行read 或 write 后就少了一次copy

```c
int fd = open(FILE_NAME, O_RDWR|O_TRUNC|O_CREAT, 0644);
for (int i = 0; i < 4; i++) {
    iov[i].iov_base = malloc(BUF_SIZE);
    iov[i].iov_len = BUF_SIZE;
    memset(iov[i].iov_base, 0, BUF_SIZE);
}
int ret = io_uring_register_buffers(ring, iov, 4);
sqe = io_uring_get_sqe(ring);
io_uring_prep_write_fixed(sqe, fd, iov[0].iov_base, str1_sz, 0, 0);
io_uring_submit(ring);
io_uring_wait_cqe(ring, &cqe);
io_uring_cqe_seen(ring, cqe);
```

注册文件

```c
int io_uring_register_files(struct io_uring *ring, const int *files , unsigned nr_files);
```

在用于提交的SQE中，您在使用文件描述符数组中的文件描述符索引而不是在像` io_uring_prep_readv（）`和`io_uring_prep_writev（）`这样的调用中使用文件描述符本身时设置了` IOSQE_FIXED_FILE ` 标志。

```c
    int fds[2];
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int ret = io_uring_register_files(ring, fds, 1);
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_write(sqe, 0, buff1, str1_sz, 0);		
    //这里的0表示注册的一组文件描述符的索引只有在设置flag |= IOSQE_FIXED_FILE 有效
    sqe->flags |= IOSQE_FIXED_FILE;	
	io_uring_submit(ring);
```



### queue polling

减少系统调用的数量是IO的一个主要目标。为此，`io_uring`允许提交I/O请求，而无需进行单个系统调用。这是通过`io_uring`支持的一个特殊的提交队列轮询特性完成的。在这种模式下，在程序设置轮询模式后，` o_uring `启动一个特殊的内核线程，该线程轮询共享提交队列中程序可能添加的条目。这样，您只需将条目提交到共享队列中，内核线程应该看到它并拾取提交队列条目，而无需您的程序进行io_uring_enter（）系统调用。这是在用户空间和内核之间共享队列的一个好处。

通过在io_uring_params结构的flags成员中设置IORING_SETUP_SQPOLL标志，可以告诉io_uring要使用此模式。如果内核线程在一段时间内没有看到任何提交，它将退出，需要再次调用io_uring_enter（）系统调用来唤醒内核线程，这里的时间由 io_uring_param 的成员 `sq_thread_idle` 所决定

在使用liburing时，您永远不会直接调用 ` io_uring_enter（）`系统调用。这通常是由` liburing `的` io_uring_submit（）`函数来处理的。它会自动判断你是否在使用轮询模式，并处理你的程序何时需要调用` io_uring_enter（）`，而你不必为此费心。

```c
    struct io_uring ring;
    struct io_uring_params params;
    memset(&params, 0, sizeof(params));
    params.flags |= IORING_SETUP_SQPOLL;			// 设置poll模式
    params.sq_thread_idle = 2000;
    io_uring_queue_init_params(8, &ring, &params);
```



### eventfd

首先我们回顾下 eventfd 系统调用

```c
int eventfd(unsigned int initval, int flags);
```

读操作：

每次成功的 read(2) 操作都会返回一个 8 字节的整数。如果提供的缓冲区大小小于 8 字节，则 read(2) 会失败并返回错误 EINVAL。

read(2) 返回的值采用主机字节序，即主机上用于整数的本机字节序。

read(2) 的行为取决于 eventfd 计数器当前是否具有非零值以及创建 eventfd 文件描述符时是否指定了 EFD_SEMAPHORE 标志：

- 如果未指定 EFD_SEMAPHORE 并且 eventfd 计数器具有非零值，则 read(2) 会返回 8 字节的数据，其中包含该值，并将计数器的值重置为零。

- 如果指定了 EFD_SEMAPHORE 并且 eventfd 计数器具有非零值，则 read(2) 会返回 8 字节的数据，其中包含值 1，并将计数器的值减 1。

- 如果在调用 read(2) 时 eventfd 计数器为零，则调用会阻塞，直到计数器变为非零（此时 read(2) 会按上述方式进行）；如果文件描述符已被设置为非阻塞，则会失败并返回错误 EAGAIN。

写操作：

write(2) 调用会将其缓冲区中提供的 8 字节整数值**添加**到计数器中。计数器中可以存储的最大值是最大无符号 64 位值减 1（即 0xfffffffffffffffe）。如果加法会导致计数器的值超过最大值，那么 write(2) 会阻塞，直到对文件描述符执行 read(2) 操作，或者如果文件描述符已被设置为非阻塞，则会失败并返回错误 EAGAIN。



liburing提供了封装

```c
int io_uring_register_eventfd(struct io_uring *ring, int fd);
```

` io_uring_register_eventfd ` 将eventfd的文件描述符fd注册到io_uring环上，当完成队列中有事件时，会对event执行write操作

如果不再需要通知，可以调用io_uring_unregister_eventfd（3）来删除eventfd注册。  不需要eventfd参数，因为一个环只能注册一个eventfd。

```c
#define BUFF_SZ   512

char buff[BUFF_SZ + 1];
struct io_uring ring;

void error_exit(char *message) {
    perror(message);
    exit(EXIT_FAILURE);
}

void *listener_thread(void *data) {
    struct io_uring_cqe *cqe;
    int efd = (int) data;
    eventfd_t v;
    printf("%s: Waiting for completion event...\n", __FUNCTION__);

    int ret = eventfd_read(efd, &v);                //首次调用会 block , 可读以为这有事件完成了
    if (ret < 0) error_exit("eventfd_read");

    printf("%s: Got completion event.\n", __FUNCTION__);

    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        fprintf(stderr, "Error waiting for completion: %s\n",
                strerror(-ret));
        return NULL;
    }
    /* Now that we have the CQE, let's process it */
    if (cqe->res < 0) {
        fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
    }
    printf("Result of the operation: %d\n", cqe->res);
    io_uring_cqe_seen(&ring, cqe);

    printf("Contents read from file:\n%s\n", buff);
    return NULL;
}

int setup_io_uring(int efd) {
    int ret = io_uring_queue_init(8, &ring, 0);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    io_uring_register_eventfd(&ring, efd);
    return 0;
}

int read_file_with_io_uring() {
    struct io_uring_sqe *sqe;

    sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    int fd = open("/etc/passwd", O_RDONLY);
    io_uring_prep_read(sqe, fd, buff, BUFF_SZ, 0);
    io_uring_submit(&ring);

    return 0;
}

int main() {
    pthread_t t;
    int efd;

    /* Create an eventfd instance */
    efd = eventfd(0, 0);            //创建eventfd
    if (efd < 0)
        error_exit("eventfd");

    /* Create the listener thread */
    pthread_create(&t, NULL, listener_thread, (void *)efd);

    sleep(2);

    /* Setup io_uring instance and register the eventfd */
    setup_io_uring(efd);

    /* Initiate a read with io_uring */
    read_file_with_io_uring();

    /* Wait for th listener thread to complete */
    pthread_join(t, NULL);

    /* All done. Clean up and exit. */
    io_uring_queue_exit(&ring);
    return EXIT_SUCCESS;
}
```







参考：

https://unixism.net/loti/index.html

[浅析开源项目之io_uring - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/361955546)

[图解原理｜Linux I/O 神器之 io_uring (qq.com)](https://mp.weixin.qq.com/s/1wZpFhwJR-LNkQm-QzFxRQ)

https://zhuanlan.zhihu.com/p/380726590

https://zhuanlan.zhihu.com/p/334658432