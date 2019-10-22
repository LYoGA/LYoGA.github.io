---
title: "Kafka初识"
date: 2019-08-06T14:53:44+08:00
lastmod: 2019-08-06T14:53:44+08:00
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

一、为什么需要消息系统

>
1. 解耦：
  - 允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束
2. 冗余：
  - 消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险
3. 扩展性：
  - 因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过程即可
4. 灵活性 & 峰值处理能力：
  - 在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃
5. 可恢复性：
  - 系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理
6. 顺序保证：
  - 在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按照特定的顺序来处理。（Kafka 保证一个 Partition 内的消息的有序性）
7. 缓冲：
  - 有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况
8. 异步通信：
  - 消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。

二、kafka架构
2.1 拓扑架构
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190806172450.png)

2.2 相关概念

>
1. producer：
  - 消息生产者，发布消息到kafka集群的终端或服务
2. broker：
  - Kafka集群包含一个或多个服务器，这种服务器被称为broker
3. topic：
  - 每条发布到kafka集群的消息属于的类别，即kafka是面向topic的
4. partition：
  - partition是物理上的概念，每个topic包含一个或多个partition，创建topic时可指定parition数量；每个partition对应于一个文件夹，该文件夹下存储该partition的数据和索引文件
5. consumer：
  - 从kafka集群中消费消息的终端或服务
6. Consumer group：
  - high-level consumer API中，每个consumer都属于一个consumer group，每条消息只能被consumer group中的一个Consumer消费，但可以被多个consumer group消费
7. replica：
  - partition的副本，保障partition的高可用
8. leader：
  - replica中的一个角色，producer和consumer只跟leader交互
9. follower：
  - replica中的一个角色，从leader中复制数据
10. controller：
  - kafka集群中的其中一个服务器，用来进行leader election以及各种failover
12. zookeeper：
  - kafka通过zookeeper来存储集群的meta信息
