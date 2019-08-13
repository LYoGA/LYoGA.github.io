---
title: "Kafka-初识"
date: 2019-08-06T14:53:44+08:00
lastmod: 2019-08-06T14:53:44+08:00
draft: true
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
1. **解耦：** 允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束
2. **冗余：** 消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险
3. **扩展性：** 因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过程即可
4. **灵活性 & 峰值处理能力：** 在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃
5. **可恢复性：** 系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理
6. **顺序保证：** 在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按照特定的顺序来处理。（Kafka 保证一个 Partition 内的消息的有序性）
7. **缓冲：** 有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况
8. **异步通信：** 消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。

二、kafka架构
2.1 拓扑架构
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190806172450.png)

2.2 相关概念

>
- **Producer：** 消息生产者，发布消息到kafka集群的终端或服务
- **Broker：** Kafka集群包含一个或多个服务器，这种服务器被称为broker
- **Topic：** 每条发布到kafka集群的消息属于的类别，即kafka是面向topic的
- **Partition：** partition是物理上的概念，每个topic包含一个或多个partition，创建topic时可指定parition数量；每个partition对应于一个文件夹，该文件夹下存储该partition的数据和索引文件
- **Consumer：** 从kafka集群中消费消息的终端或服务
- **Consumer group：** high-level consumer API中，每个consumer都属于一个consumer group，每条消息只能被consumer group中的一个Consumer消费，但可以被多个consumer group消费
- **Replica：** partition的副本，保障partition的高可用
- **Leader：** replica中的一个角色，producer和consumer只跟leader交互
- **Follower：** replica中的一个角色，从leader中复制数据
- **Controller：** kafka集群中的其中一个服务器，用来进行leader election以及各种failover
- **Zookeeper：** kafka通过zookeeper来存储集群的meta信息

主题与分区

主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题
同一主题下的不同分区包含的消息是不同的，分区在存储层面可以看作一个可追加的日志(Log)文件，消息在被追加到分区日志、文件的时 候都会分配一个特定的偏移量(offset)
offset是消息在分区中的唯一标识，Kafka通过它来保证消息在分区内的顺序性，不过offset并不跨越分区，也就是说，Kafka保证的是分区有序而不是主题有序

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813202501.png)

 如图所示，主题中有4个分区，消息被顺序追加到每个分区日志文件的尾部。Kafka中的分区可以分布在不同的服务器(broker)上，一个主题可以横跨多个broker，以此来提供比单个broker更强大的性能。
 如果一个主题只对应一个文件，那么这个文件所在的机器 I/O 将会成为这个主题的性能瓶颈，而分区解决了这个问题

 副本机制
 Kafka 为分区引入了多副本 (Replica) 机制， 通过增加副本数量可以提升容灾能力。同一 分区的不同副本中保存的是相同的消息(在同一时刻，副本之间并非完全一样)，自1J本之间是 “一主多从”的关系，其中 leader副本负责处理读写请求， follower副本只负责与 leader副本的 消息同步。副本处于不同的 broker 中 ，当 leader 副本出现故障时，从 follower 副本中重新选举 新的 leader副本对外提供服务。 Kafka通过多副本机制实现了故障的自动转移，当 Kafka集群中
某个 broker 失效时仍然能保证服务可用 。
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813203757.png)

如图所示，Kafka集群中有4个broker，某个主题中有3个分区，且副本因子(即副本个数)也为3，如此每个分区便有1个leader副本和2个follower副本。生产者和消费者只与leader副本进行交互，而followr副本只负责消息的同步，很多时候follower副本中的消息相对leader副本而言会有一定的滞后。

Kafka消费端也具备一定的容灾能力：Consumer使用拉(Pull)模式从服务端拉取消息，并且保存消费的具体位置，当消费者看机后恢复上线时可以根据之前保存的消费位置重新拉取需要的消息进行消费，这样就不会造成消息丢失

AR(Assigned Replicas)：分区中的所有副本统称为AR
ISR(In-Sync Replicas)：所有与 leader 副本保持 一定程度 同步 的副本(包括 leader 副本在内〕组成 ISR
OSR(Out-of-Sync Replicas)：与 leader 副本同 步滞后过 多的副本(不包 括 leader 副本)组成 OSR
AR = ISR + OSR

leader副本负责维护和跟踪ISR集合中所有follower副本的滞后状态，当follower副本落后太多或失效时，leader副本会把它从ISR集合中剔除。
如果OSR集合中有follower副本“追上”了leader副本，那么leader副本会把它从OSR集合转移至ISR集合。
默认情况下，当leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader，而在OSR集合中的副本则没有任何机会。

HW(High Watermark)：俗称高水位，它标识了一个特定的消息偏移量(offset)，消费者只能拉取到这个offset之前的消息
LEO(Log End Offset)：标识当前日志文件中下一条待写入消息的offset

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813204921.png)
如图 1-4 所示，它代表一个日志文件，这个日志文件中有 9 条消息，第一条消息的 offset (LogStartOffset)为 0，最后一条消息的 offset为 8, offset为 9 的消息用虚线框表示，代表下 一条待写入 的消息 。日志文件的 HW 为 6，表示消费者只能拉取到 offset 在 0 至 5 之间的消息， 而 offset 为 6 的消息对消 费者而言是不可见 的 。
分区 ISR集合中的每个副本都会维护自身的 LEO，而ISR集合中最小的 LEO即为分区的 HW ，对消费者而言只能消费 HW 之前的消息 。
