---
title: "Java NIO -- Buffer"
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
Java NIO中的Buffer用于和NIO通道进行交互。如你所知，数据是从通道读入缓冲区，从缓冲区写入到通道中的。
缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存，内部使用字节数组存储数据，并维护几个特殊变量，实现数据的反复利。
这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。

```Java
public abstract class Buffer {
    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;
}
```
1. mark：初始值为-1，用于备份当前的position
2. position：初始值为0，position表示当前可以写入或读取数据的位置，当写入或读取一个数据后，position向前移动到下一个位置.
3. limit：写模式下，limit表示最多能往Buffer里写多少数据，等于capacity值；读模式下，limit表示最多可以读取多少数据
4. capacity：缓存数组大小
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20180828152802.png)

## **二、相关方法** ##
```Java
// 把当前的position赋值给mark
public final Buffer mark() {
        mark = position;
        return this;
}

// 把mark值还原给position
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}

// 一旦读完Buffer中的数据，需要让Buffer准备好再次被写入，clear会恢复状态值，但不会擦除数据
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
// compact()：compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据

// Buffer有两种模式，写模式和读模式，flip后Buffer从写模式变成读模式
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}

// 重置position为0，从头读写数据
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

**Buffer的分配**
```
ByteBuffer buf = ByteBuffer.allocate(24);
```

**向Buffer里面写数据**

- 从Channel写到Buffer
- 通过Buffer的put()方法写到Buffer里

从Channel写到Buffer的例子
```
// read into buffer.
// 对于Buffer来说，们要将来自Channel的数据填充到Buffer中，在系统层面上，这个操作我们称为读操作，因为数据是从外部（文件或网络等）读到内存中。
int bytesRead = inChannel.read(buf);
```
通过put方法写Buffer的例子：
```
buf.put(127);
// put方法有很多版本，允许你以不同的方式把数据写入到Buffer中。例如，写到一个指定的位置，或者把一个字节数组写入到Buffer；更多Buffer实现的细节参考JavaDoc。
```

**从Buffer中读取数据**

- 从Buffer读取数据到Channel
- 使用get()方法从Buffer中读取数据

从Buffer读取数据到Channel的例子：
```
// read from buffer into channel.
// 将写入的数据传输到Channel中，如通过FileChannel将数据写入到文件中，通过SocketChannel将数据写入网络发送到远程机器等。对应的，这种操作，我们称之为写操作。
int bytesWritten = inChannel.write(buf);
```
使用get()方法从Buffer中读取数据的例子
```
byte aByte = buf.get();
// get方法有很多版本，允许你以不同的方式从Buffer中读取数据。例如，从指定position读取，或者从Buffer中读取数据到字节数组；更多Buffer实现的细节参考JavaDoc。
```
**相关方法测试用例**
```
public static void main(String[] args) {
    // 1.使用allocate()申请10个字节的缓冲区
    ByteBuffer byteBuffer = ByteBuffer.allocate(10);
    System.out.println("------------allocate------------------");
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    // 2.使用put()存放5个字节到缓冲区
    byteBuffer.put("abcdefghij".getBytes());
    System.out.println("------------put------------------");
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    // 3.切换到读取数据模式
    byteBuffer.flip();
    System.out.println("------------flip------------------");
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    // 4.从缓冲区中读取数据
    System.out.println("------------get------------------");
    byte[] bytes = new byte[byteBuffer.limit()];
    byteBuffer.get(bytes);
    System.out.println(new String(bytes, 0, bytes.length));
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    // 5.设置为可重复读取
    System.out.println("------------rewind------------------");
    byteBuffer.rewind();
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());
    byte[] bytes2 = new byte[byteBuffer.limit()];
    byteBuffer.get(bytes2);
    System.out.println(new String(bytes2, 0, bytes2.length));
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    // 6.clear清空缓存区，但是内容没有被清掉，还存在。只不过这些数据状态为被遗忘状态。
    System.out.println("------------clear------------------");
    byteBuffer.clear();
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());
    byte[] bytes3 = new byte[10];
    byteBuffer.get(bytes3);
    System.out.println(new String(bytes3, 0, bytes3.length));

    byteBuffer.clear();
    byteBuffer.put("hijkl".getBytes());
    byteBuffer.flip();
    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    byte[] bytes4 = new byte[byteBuffer.limit()];
    byteBuffer.get(bytes4, 0, 2);
    System.out.println(new String(bytes4, 0, bytes4.length));

    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    byteBuffer.mark();
    System.out.println("------------mark------------------");

    byteBuffer.get(bytes4, 0, 2);
    System.out.println(new String(bytes4, 0, bytes4.length));

    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());

    byteBuffer.reset();
    System.out.println("------------reset------------------");

    System.out.println(byteBuffer.position());
    System.out.println(byteBuffer.limit());
    System.out.println(byteBuffer.capacity());
}
```

## **三、相关实现类** ##
目前Buffer的实现类有以下几种：

- ByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer
- MappedByteBuffer
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20180828165951.png)

**ByteBuffer**

ByteBuffer的实现类包括HeapByteBuffer和DirectByteBuffer两种

**HeapByteBuffer**
```
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}
// 在虚拟机堆上申请内存空间
HeapByteBuffer(int cap, int lim) {  
    super(-1, 0, lim, cap, new byte[cap], 0);
}
```

**DirectByteBuffer**
```
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}

DirectByteBuffer(int cap) {
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```
DirectByteBuffer通过unsafe.allocateMemory申请堆外内存，并在ByteBuffer的address变量中维护指向该内存的地址。
unsafe.setMemory(base, size, (byte) 0)方法把新申请的内存数据清零

## **参考** ##
- https://www.jianshu.com/p/052035037297
- http://ifeve.com/buffers/
