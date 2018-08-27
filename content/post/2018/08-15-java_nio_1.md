---
title: "Java NIO -- Selector"
date: 2018-08-15T14:52:54+08:00
lastmod: 2018-08-15T14:52:54+08:00
draft: false
keywords:
- Java NIO
description: ""
tags:
- Java
- Java NIO
categories:
- Java NIO
author: "lyoga"

---

<!--more-->
## **前言** ##
Java NIO由以下几个核心部分组成：

1. Buffer
2. Channel
3. Selector


## **一、概念** ##
Selector（选择器）是Java NIO中能够检测一到多个NIO通道，并能够知晓通道是否为比如读写事件做好准备的组件。这样一个单独的线程可以管理多个channel，从而管理多个网络连接。

**通俗理解：** 在一个养鸡场，有这么一个人，每天的工作就是不停检查几个特殊的鸡笼，如果有鸡进来，有鸡出去，有鸡生蛋，有鸡生病等等，就把相应的情况记录下来，如果鸡场的负责人想知道情况，只需要询问那个人即可。
在这里，这个人就相当Selector，每个鸡笼相当于一个SocketChannel，每个线程通过一个Selector可以管理多个SocketChannel。
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20180823192237.png)


## **二、Selector原理** ##
为了实现Selector管理多个SocketChannel，必须将具体的SocketChannel对象注册到Selector，并声明需要监听的事件（这样Selector才知道需要记录什么数据）

**监听事件：**

1. **connect：** 客户端连接服务端事件，对应值为 **SelectionKey.OP_CONNECT(8)**
2. **accept：** 服务端接收客户端连接事件，对应值为 **SelectionKey.OP_ACCEPT(16)**
3. **read：** 读事件，对应值为 **SelectionKey.OP_READ(1)**
4. **write：** 写事件，对应值为 **SelectionKey.OP_WRITE(4)**

因此Selector需要其他两种对象配合使用，即SelectionKey和SelectableChannel，下图表明三者的关系：
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20180823193724.png)

- **SelectableChannel：** 可以与Selector进行配合的通道，例如Socket相关通道以及Pipe产生的通道都属于SelectableChannel。这类通道可以将自己感兴趣的操作（即上述监听事件）注册到一个Selector上，并在Selector的控制下进行IO相关操作
- **Selector：** 负责管理已注册的多个SelectableChannel，当这些通道的某些状态改变时，Selector会被唤醒（从select()方法的阻塞中），并对所有就绪的通道进行轮询操作
- **SelectionKey：** 用来记录SelectableChannel和Selector之间关系的对象，它由SelectableChannel的register()方法返回，并存储在Selector的多个集合中。它不仅记录了两个对象的引用，还包含了SelectableChannel感兴趣的操作

```Java
// 创建一个ServerSocketChannel
ServerSocketChannel serverChannel = ServerSocketChannel.open();
// 通道定义为非阻塞
serverChannel.configureBlocking(false);
// 绑定端口
serverChannel.socket().bind(new InetSocketAddress(port));
// 创建一个Selector
Selector selector = Selector.open();
// 将serverSocketChannel注册到selector，并指定事件OP_ACCEPT
serverChannel.register(selector, SelectionKey.OP_ACCEPT);
while(true){
    int n = selector.select();
    if (n == 0) continue;
    Iterator ite = this.selector.selectedKeys().iterator();
    while(ite.hasNext()){
        SelectionKey key = (SelectionKey)ite.next();
        //如果满足Acceptable条件，则必定是一个ServerSocketChannel
        if (key.isAcceptable()){
            //得到一个连接好的SocketChannel，并把它注册到Selector上，兴趣操作为READ，同时为该信道指定关联的附件
            SocketChannel clntChan = ((ServerSocketChannel) key.channel()).accept();
            clntChan.configureBlocking(false);
            clntChan.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocate(bufSize));
        }
        if (key.isReadable()){
            handleRead(key);
        }
        if (key.isWritable() && key.isValid()){
            handleWrite(key);
        }
        if (key.isConnectable()){
            System.out.println("isConnectable = true");
        }
      ite.remove();
    }
}
```

### **2.1 SelectionKey** ###
一个SelectionKey表示了一个特定的通道对象和一个特定的选择器对象之间的注册关系。
```Java
// 属性
final SelChImpl channel;                            
public final SelectorImpl selector;
private volatile int interestOps; // interest集合代表监听事件集合
private int readyOps; // 通道已经准备就绪的操作的集合
private volatile Object attachment = null; // 附加对象，可以方便识别某个给定对象
```

**interestOps**
```Java
int interestSet = selectionKey.interestOps();
boolean isInterestedInAccept  = (interestSet & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT；
boolean isInterestedInConnect = (interestSet & SelectionKey.OP_CONNECT) == SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = (interestSet & SelectionKey.OP_READ) == SelectionKey.OP_READ;
boolean isInterestedInWrite   = (interestSet & SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE;
```
可以看到，用“位与”操作interest 集合和给定的SelectionKey常量，可以确定某个确定的事件是否在interest 集合中。

**readyOps**
```Java
int readySet = selectionKey.readyOps();
//检查这些操作是否就绪的方法
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

### **2.2 Selector三个集合** ###
- **all-keys集合：** 当前所有向Selector注册的SelectionKey的集合，Selector的 **keys()** 方法返回该集合,并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException
- **selected-keys集合：** 相关事件已经被Selector捕获的SelectionKey的集合，Selector的 **selectedKeys()** 方法返回该集合，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException
- **cancelled-keys集合：** 已经被取消的SelectionKey的集合，Selector没有提供访问这种集合的方法

### **2.3 Selector.open()** ###
SocketChannel、ServerSocketChannel和Selector的实例初始化都通过SelectorProvider类实现，其中Selector是整个NIO Socket的核心实现
```Java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}

//SelectorProvider在windows和linux下有不同的实现，provider方法会返回对应的实现
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        //在与当前线程相同访问控制权限的环境中，加载SelectorProvider实例    
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                        //获取系统配置的SelectorProvider    
                        if (loadProviderFromProperty())
                            return provider;
                        //获取类加载路径下的SelectorProvider    
                        if (loadProviderAsService())
                            return provider;
                        //加载默认的SelectorProvider    
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
```

以下主要以windows的实现来梳理整个流程
```Java
// WindowsSelectorProvider.java  
public AbstractSelector openSelector() throws IOException {  
    return new WindowsSelectorImpl(this);  
}  

// WindowsSelectorImpl.java  
WindowsSelectorImpl(SelectorProvider sp) throws IOException {  
    super(sp);  
    pollWrapper = new PollArrayWrapper(INIT_CAP);  
    wakeupPipe = Pipe.open();  
    wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();  

    // Disable the Nagle algorithm so that the wakeup is more immediate  
    SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();  
    (sink.sc).socket().setTcpNoDelay(true);  
    wakeupSinkFd = ((SelChImpl)sink).getFDVal();  

    pollWrapper.addWakeupSocket(wakeupSourceFd, 0);  
}  
```
- 创建pollWrapper对象
- Pipe.open()打开一个管道
- 拿到wakeupSourceFd(从管道读数据)和wakeupSinkFd(往管道写数据)两个文件描述符
- 把唤醒端的文件描述符（wakeupSourceFd）放到pollWrapper里

```Java
// PollArrayWrapper
typedef struct pollfd {
  SOCKET fd;            // 4 bytes
  short events;         // 2 bytes
} pollfd_t;

class PollArrayWrapper {

    private AllocatedNativeObject pollArray; // The fd array
    long pollArrayAddress; // pollArrayAddress

    @Native private static final short FD_OFFSET     = 0; // fd offset in pollfd
    @Native private static final short EVENT_OFFSET  = 4; // events offset in pollfd

    static short SIZE_POLLFD = 8; // sizeof pollfd struct

    private int size; // Size of the pollArray

    PollArrayWrapper(int newSize) {
        int allocationSize = newSize * SIZE_POLLFD;
        pollArray = new AllocatedNativeObject(allocationSize, true);
        pollArrayAddress = pollArray.address();
        this.size = newSize;
    }
}
```
**pollWrapper用Unsafe类申请一块物理内存pollfd，存放socket句柄fdVal和events，其中pollfd共8位，0-3位保存socket句柄，4-7位保存events**

pollWrapper提供了fdVal和event数据的相应操作，如添加操作通过Unsafe的putInt和putShort实现
```Java
void putDescriptor(int i, int fd) {
    pollArray.putInt(SIZE_POLLFD * i + FD_OFFSET, fd);
}

void putEventOps(int i, int event) {
    pollArray.putShort(SIZE_POLLFD * i + EVENT_OFFSET, (short)event);
}

// Adds Windows wakeup socket at a given index.
//这里将source的POLLIN事件标识为感兴趣的，当sink端有数据写入时，source对应的文件描述符wakeupSourceFd就会处于就绪状态
void addWakeupSocket(int fdVal, int index) {
    putDescriptor(index, fdVal);
    putEventOps(index, Net.POLLIN);
}
```

### **2.4 Channel.register()** ###
我们来看看serverChannel.register(selector, SelectionKey.OP_ACCEPT)是如何实现的
```Java
public final SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
            }
            return k;
        }
    }
```
1. 如果该channel和selector已经注册过，则直接添加事件和附件
2. 否则通过selector实现注册过程

```Java
protected final SelectionKey register(AbstractSelectableChannel ch,
      int ops,  Object attachment) {
    if (!(ch instanceof SelChImpl))
        throw new IllegalSelectorException();
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);
    synchronized (publicKeys) {
        implRegister(k);
    }
    k.interestOps(ops);
    return k;
}

protected void implRegister(SelectionKeyImpl ski) {
    synchronized (closeLock) {
        if (pollWrapper == null)
            throw new ClosedSelectorException();
        growIfNeeded();
        channelArray[totalChannels] = ski;
        ski.setIndex(totalChannels);
        fdMap.put(ski);
        keys.add(ski);
        pollWrapper.addEntry(totalChannels, ski);
        totalChannels++;
    }
}
```
1. 以当前channel和selector为参数，初始化SelectionKeyImpl对象selectionKeyImpl，并添加附件attachment
2. 如果当前channel的数量totalChannels等于SelectionKeyImpl数组大小，对SelectionKeyImpl数组和pollWrapper进行扩容操作
3. 如果totalChannels % MAX_SELECTABLE_FDS == 0，则多开一个线程处理selector
4. pollWrapper.addEntry将把selectionKeyImpl中的socket句柄添加到对应的pollfd
5. k.interestOps(ops)方法最终也会把event添加到对应的pollfd

**因此，无论是serverSocketChannel，还是socketChannel，在selector注册的事件，最终都保存在pollArray中**

### **2.5 Selector.select()** ###
通过Selector的 **select()** 方法可以选择已经准备就绪的通道(这些通道包含你感兴趣的的事件)

下面是Selector几个重载的select()方法：

- int select()：阻塞到至少有一个通道在你注册的事件上就绪了
- int select(long timeout)：和select()一样，但最长阻塞时间为timeout毫秒
- int selectNow()：非阻塞，只要有通道就绪就立刻返回

**select()方法返回的int值表示有多少通道已经就绪，是自上次调用select()方法后有多少通道变成就绪状态。之前在select()调用时进入就绪的通道不会在本次调用中被记入，而在前一次select()调用进入就绪但现在已经不在处于就绪的通道也不会被记入。**

例如：首次调用select()方法，如果有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道就绪了。

现在我们来看看selector中的select是如何实现一次获取多个有事件发生的channel的，底层由selector实现类的doSelect方法实现，如下：
```Java
protected int doSelect(long timeout) throws IOException {
        if (channelArray == null)
            throw new ClosedSelectorException();
        this.timeout = timeout; // set selector timeout
        processDeregisterQueue();
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        // Calculate number of helper threads needed for poll. If necessary
        // threads are created here and start waiting on startLock
        adjustThreadsCount();
        finishLock.reset(); // reset finishLock
        // Wakeup helper threads, waiting on startLock, so they start polling.
        // Redundant threads will exit here after wakeup.
        startLock.startThreads();
        // do polling in the main thread. Main thread is responsible for
        // first MAX_SELECTABLE_FDS entries in pollArray.
        try {
            begin();
            try {
                subSelector.poll();
            } catch (IOException e) {
                finishLock.setException(e); // Save this exception
            }
            // Main thread is out of poll(). Wakeup others and wait for them
            if (threads.size() > 0)
                finishLock.waitForHelperThreads();
          } finally {
              end();
          }
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        finishLock.checkForException();
        processDeregisterQueue();
        int updated = updateSelectedKeys();
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        resetWakeupSocket();
        return updated;
    }

    private final class SubSelector {
       // These arrays will hold result of native select().
       // The first element of each array is the number of selected sockets.
       // Other elements are file descriptors of selected sockets.
       private final int[] readFds = new int [MAX_SELECTABLE_FDS + 1];
       private final int[] writeFds = new int [MAX_SELECTABLE_FDS + 1];
       private final int[] exceptFds = new int [MAX_SELECTABLE_FDS + 1];

       private int poll() throws IOException{ // poll for the main thread
           return poll0(pollWrapper.pollArrayAddress,
                        Math.min(totalChannels, MAX_SELECTABLE_FDS),
                        readFds, writeFds, exceptFds, timeout);
       }
     }
```

- **subSelector.poll()** 是select的核心，由native函数poll0实现，readFds、writeFds 和exceptFds数组用来保存底层select的结果，数组的第一个位置都是存放发生事件的socket的总数，其余位置存放发生事件的socket句柄fd。
- 执行 selector.select() ，poll0函数把指向socket句柄和事件的内存地址传给底层函数。
  1. 如果之前没有发生事件，程序就阻塞在select处，当然不会一直阻塞，因为epoll在timeout时间内如果没有事件，也会返回
  2. 一旦有对应的事件发生，poll0方法就会返回
  3. processDeregisterQueue方法会清理那些已经cancelled的SelectionKey
  4. updateSelectedKeys方法统计有事件发生的SelectionKey数量，并把符合条件发生事件的SelectionKey添加到selectedKeys哈希表中，提供给后续使用

在早期的JDK1.4和1.5 update10版本之前，Selector基于select/poll模型实现，是基于IO复用技术的非阻塞IO，不是异步IO;在JDK1.5 update10和linux core2.6以上版本，sun优化了Selctor的实现，底层使用epoll替换了select/poll。

### **2.6 Selector.wakeUp()** ###

1. 某个线程调用select()方法后阻塞了，即使没有通道已经就绪，只要让其它线程在第一个线程调用select()方法的那个对象上调用Selector.wakeup()方法即可，阻塞在select()方法上的线程会立马返回
2. 如果有其它线程调用了wakeup()方法，但当前没有线程阻塞在select()方法上，下个调用select()方法的线程会立即“醒来（wake up）”

```
public Selector wakeup() {
    synchronized (interruptLock) {
        if (!interruptTriggered) {
            setWakeupSocket();
            interruptTriggered = true;
        }
    }
    return this;
}

// Sets Windows wakeup socket to a signaled state.
private void setWakeupSocket() {
   setWakeupSocket0(wakeupSinkFd);
}

private native void setWakeupSocket0(int wakeupSinkFd);
JNIEXPORT void JNICALL  
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,  
                                                jint scoutFd)  
{  
    /* Write one byte into the pipe */  
    const char byte = 1;  
    send(scoutFd, &byte, 1, 0);  
}  
```
可见wakeup()是通过pipe的write端send(scoutFd, &byte, 1, 0)，发送一个字节1，来唤醒poll(),所以在需要的时候就可以调用selector.wakeup()来唤醒selector


### **2.7 Selector.close()** ###

该方法使得任何一个在选择操作中阻塞的线程都被唤醒（类似wakeup()），同时使得注册到该Selector的所有Channel被注销，所有的键将被取消，但是Channel本身并不会关闭。

## **三、NIO轶事** ##
Java selector.open() 的时候，会创建一个自己和自己的链接（windows上是tcp，linux上是通道pipe），这是为什么呢？
这么做的原因可以从Apache Mina中窥探。

在Mina中，有如下机制：

1. Mina框架会创建一个Work对象的线程。
2. Work对象的线程的run()方法会从一个队列中拿出一堆Channel，然后使用Selector.select()方法来侦听是否有数据可以读/写。
3. 最关键的是，在select的时候，如果队列有新的Channel加入，那么，Selector.select()会被唤醒，然后重新select最新的Channel集合。
4. 要唤醒select方法，只需要调用Selector的wakeup()方法。

一个阻塞在select上的线程有以下三种方式可以被唤醒：

1. 有数据可读/写，或出现异常。
2. 阻塞时间到，即time out。
3. 收到一个non-block的信号。可由kill或pthread_kill发出。
所以，Selector.wakeup()要唤醒阻塞的select，那么也只能通过这三种方法，其中：

第二种方法可以排除，因为select一旦阻塞，应无法修改其time out时间；而第三种看来只能在Linux上实现，Windows上没有这种信号通知的机制。
所以看来只有第一种方法了。

因此Java NIO为什么要创建一个自己和自己的链接：如果想要唤醒select，只需要朝着自己的这个loopback连接发点数据过去，于是就可以唤醒阻塞在select上的线程了

## **参考** ##
- https://www.jianshu.com/p/0d497fe5484a
- http://ifeve.com/selectors/
- https://www.cnblogs.com/snailclimb/p/9086334.html
- http://goon.iteye.com/blog/1764057
- http://zhhphappy.iteye.com/blog/2032893
