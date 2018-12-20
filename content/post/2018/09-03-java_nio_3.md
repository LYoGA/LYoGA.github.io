---
title: "Java NIO -- Channel"
date: 2018-09-03T19:58:47+08:00
lastmod: 2018-09-03T19:58:47+08:00
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
Java NIO的通道类似流，但又有些不同：

1. 既可以从Channel中读取数据，又可以写数据到Channel，但Stream的读写通常是单向的
2. Channel可以异步地读写，而Stream是阻塞的同步读写
3. Channel中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入

正如上面所说，从通道读取数据到缓冲区，从缓冲区写入数据到通道。如下图所示：
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20181017172855.png)

## **二、实现** ##
这些是Java NIO中最重要的通道的实现：

- FileChannel : 从文件中读写数据
- DatagramChannel : 能通过UDP读写网络中的数据
- SocketChannel : 能通过TCP读写网络中的数据
- ServerSocketChannel : 可以监听新进来的TCP连接，像Web服务器那样,对每一个新进来的连接都会创建一个SocketChannel
