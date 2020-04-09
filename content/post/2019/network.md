---
title: "Network"
date: 2020-01-05T19:31:46+08:00
lastmod: 2020-01-05T19:31:46+08:00
draft: false
keywords:
-
description: ""
tags:
-
categories:
-
author: "lyoga"
---

<!--more-->
# **Linux IO模式**
**当一个read操作发生时，它会经历两个阶段：**

1. 等待数据准备 (Waiting for the data to be ready)
2. 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)

## **IO模式**

- 阻塞 I/O（blocking IO）
- 非阻塞 I/O（nonblocking IO）
- I/O 多路复用（ IO multiplexing）
- 信号驱动 I/O（ signal driven IO）
- 异步 I/O（asynchronous IO）

## **阻塞I/O**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200105203215.png)
**blocking IO的特点就是在IO执行的两个阶段都被block了**

## **非阻塞I/O**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200105203248.png)
**nonblocking IO的特点是用户进程需要不断的主动询问kernel数据好了没有**

## **I/O多路复用**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200105203326.png)

- IO multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO
- 当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回
- I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回
- 在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block

## **异步I/O**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200105203501.png)
**用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了**

## **总结**
### **blocking和non-blocking的区别**
调用blocking IO会一直block住对应的进程直到操作完成，而non-blocking IO在kernel还准备数据的情况下会立刻返回

### **synchronous IO和asynchronous IO的区别**
**在说明synchronous IO和asynchronous IO的区别之前，需要先给出两者的定义。POSIX的定义是这样子的：**

- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
- An asynchronous I/O operation does not cause the requesting process to be blocked;

**两者的区别就在于synchronous IO做”IO operation”的时候会将process阻塞。按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO**

- 定义中所指的”IO operation”是指真实的IO操作，就是例子中的recvfrom这个system call
- non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程
- 但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的
- 而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200105203844.png)

# **I/O 多路复用之select、poll、epoll详解**
## **select**
```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

**select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理，缺点主要有如下三个方面：**

- select最大的缺陷就是单个进程所打开的FD是有一定限制的，它由FD_SETSIZE设置，默认值是1024
- 对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低
- 需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大

## **poll**
```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);

struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```
**不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现**

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符

**相比select，poll的最大优点是没有最大连接数的限制，其原因是它是基于链表来存储的，但是同样有缺点：**

- 大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制是不是有意义
- poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd

> 从上面看，select和poll都需要在返回后，**通过遍历文件描述符来获取已经就绪的socket**。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降

## **epoll**
```
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)； //对指定描述符fd执行op操作
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout); //等待epfd上的io事件，最多返回maxevents个事件
```
**内部用了一个红黑树记录添加的socket，用了一个双向链表接收内核触发的事件**

- 执行epoll_create时，创建了红黑树和就绪链表
- 执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据
- 执行epoll_wait时立刻返回准备就绪链表里的数据即可

**在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在)**

## **优点**

- 没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）
- 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；即epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll
- 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递，即epoll使用mmap减少复制开销

## **参考**

- https://segmentfault.com/a/1190000003063859
- https://blog.csdn.net/tianjing0805/article/details/76021440

# **Proactor 与 Reactor**
**Reactor模式用于同步I/O，而Proactor运用于异步I/O操作**

## **Reactor模型**

1. 向事件分发器注册事件回调
2. 事件发生
3. 事件分发器调用之前注册的函数
4. 在回调函数中读取数据，对数据进行后续处理

**Reactor模型实例：libevent，Redis、ACE**

## **Proactor模型**

1. 向事件分发器注册事件回调
2. 事件发生
3. 操作系统读取数据，并放入应用缓冲区，然后通知事件分发器
4. 事件分发器调用之前注册的函数
5. 在回调函数中对数据进行后续处理

**Preactor模型实例：ASIO**

## **reactor和proactor的主要区别**

### **主动和被动**
**1. 以主动写为例**

- Reactor将handle放到select()，等待可写就绪，然后调用write()写入数据；写完处理后续逻辑
- Proactor调用aoi_write后立刻返回，由内核负责写操作，写完后调用相应的回调函数处理后续逻辑

**可以看出，Reactor被动的等待指示事件的到来并做出反应；它有一个等待的过程，做什么都要先放入到监听事件集合中等待handler可用时再进行操作；
Proactor直接调用异步读写操作，调用完后立刻返回**

**2. 实现**

- Reactor实现了一个被动的事件分离和分发模型，服务等待请求事件的到来，再通过不受间断的同步处理事件，从而做出反应；
- Proactor实现了一个主动的事件分离和分发模型；这种设计允许多个任务并发的执行，从而提高吞吐量；并可执行耗时长的任务（各个任务间互不影响）

## **参考**

- https://www.cnblogs.com/losophy/p/9202815.html
