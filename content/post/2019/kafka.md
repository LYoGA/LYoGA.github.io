---
title: "Kafka-初识"
date: 2019-08-06T14:53:44+08:00
lastmod: 2019-08-06T14:53:44+08:00
draft: false
keywords:
-
description: ""
tags:
- kafka
categories:
- kafka
author: "lyoga"
---

<!--more-->
# **一、为什么需要消息系统**#
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

&nbsp;

# **二、kafka架构** #
## **2.1 拓扑架构** ##
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190806172450.png)

&nbsp;

## **2.2 相关概念** ##

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

&nbsp;

# **三、主题与分区** #

>
- 主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题
- 同一主题下的不同分区包含的消息是不同的，分区在存储层面可以看作一个可追加的日志(Log)文件，消息在被追加到分区日志文件的时候都会分配一个特定的偏移量(offset)
- offset是消息在分区中的唯一标识，Kafka通过它来保证消息在分区内的顺序性，不过offset并不跨越分区，也就是说，Kafka保证的是分区有序而不是主题有序

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813202501.png)

 > 如图所示，主题中有4个分区，消息被顺序追加到每个分区日志文件的尾部。Kafka中的分区可以分布在不同的服务器(broker)上，一个主题可以横跨多个broker，以此来提供比单个broker更强大的性能。
 如果一个主题只对应一个文件，那么这个文件所在的机器 I/O 将会成为这个主题的性能瓶颈，而分区解决了这个问题

&nbsp;

 **副本机制**

- Kafka为分区引入了多副本(Replica)机制，通过增加副本数量可以提升容灾能力
- 同一分区的不同副本中保存的是相同的消息(在同一时刻，副本之间并非完全一样)，副本之间是 “一主多从”的关系，其中leade副本负责处理读写请求，follower副本只负责与leader副本的消息同步
- 副本处于不同的broker中，当leader副本出现故障时，从follower副本中重新选举新的leader副本对外提供服务
- Kafka通过多副本机制实现了故障的自动转移，当Kafka集群中某个broker失效时仍然能保证服务可用

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813203757.png)

> 如图所示，Kafka集群中有4个broker，某个主题中有3个分区，且副本因子(即副本个数)也为3，如此每个分区便有1个leader副本和2个follower副本。
生产者和消费者只与leader副本进行交互，而followr副本只负责消息的同步，很多时候follower副本中的消息相对leader副本而言会有一定的滞后。


Kafka消费端也具备一定的容灾能力：Consumer使用拉(Pull)模式从服务端拉取消息，并且保存消费的具体位置，当消费者看机后恢复上线时可以根据之前保存的消费位置重新拉取需要的消息进行消费，这样就不会造成消息丢失

>
- AR(Assigned Replicas)：分区中的所有副本统称为AR
- ISR(In-Sync Replicas)：所有与leader副本保持 一定程度 同步 的副本(包括leader副本在内)组成ISR
- OSR(Out-of-Sync Replicas)：与leader副本同步滞后过多的副本(不包括leader副本)组成OSR
> **AR = ISR + OSR**

- leader副本负责维护和跟踪ISR集合中所有follower副本的滞后状态，当follower副本落后太多或失效时，leader副本会把它从ISR集合中剔除
- 如果OSR集合中有follower副本“追上”了leader副本，那么leader副本会把它从OSR集合转移至ISR集合
- 默认情况下，当leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader，而在OSR集合中的副本则没有任何机会

&nbsp;

**HW(High Watermark)：** 俗称高水位，它标识了一个特定的消息偏移量(offset)，消费者只能拉取到这个offset之前的消息

**LEO(Log End Offset)：** 标识当前日志文件中下一条待写入消息的offset

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190813204921.png)

> 如图所示，代表一个日志文件，这个日志文件中有9条消息，第一条消息的offset(LogStartOffset)为0，最后一条消息的offset为 8,offset为9的消息用虚线框表示，代表下一条待写入的消息。日志文件的HW为6，表示消费者只能拉取到offset在0至5之间的消息，而offset为6的消息对消费者而言是不可见的

**分区ISR集合中的每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW，对消费者而言只能消费HW之前的消息**

&nbsp;

# **四、日志存储** #
## **4.1 文件目录布局** ##

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814111455.png)

>
- 向Log中追加消息时是顺序写入的，只有最后一个LogSegment才能执行写入操作，在此之前所有的LogSegment都不能写入数据。为了方便描述，我们将最后一个LogSegment称为“activeSegment”，即表示当前活跃的日志分段。随着消息的不断写入，当activeSegment满足一定的条件时，就需要创建新的activeSegment，之后追加的消息将写入新的activeSegment
- 为了便于消息的检索，每个LogSegment中的日志文件(以“.log”为文件后缀)都有对应的两个索引文件：偏移量索引文件(以“.index”为文件后缀)和时间戳索引文件(以“.timeindex” 为文件后缀)
- 每个LogSegment都有一个基准偏移量baseOffset，用来表示当前LogSegment中第一条消息的offset
- 偏移量是一个64位的长整型数，日志文件和两个索引文件都是根据基准偏移量(baseOffset)命名的，名称固定为20位数字，没有达到的位数则用0填充。比如第一个LogSegment的基准偏移量为0，对应的日志文件为000000000000000000000.log

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814112438.png)

&nbsp;

## **4.2 日志格式的演变** ##
**从0.8.x版本开始到现在的2.0.0版本，Kafka的消息格式也经历了3个版本: vO版本、v1版本和v2版本**

### **4.2.1 v0版本** ###
**(0.10.0之前)**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814112840.png)

- crc32(4B):crc32校验值。校验范围为magic至value之间
- magic(1B):消息格式版本号，此版本的magic值为0
- attributes(1B):消息的属性。总共占1个字节,低3位表示压缩类型: 0表示NONE、1表示GZIP、2表示SNAPPY、3表示LZ4(LZ4自Kafka0.9.x引入)，其余位保留
- key length(4B) : 表示消息的key的长度。如果为-1，则表示没有设置key，即key = null
- key:可选，如果没有key则无此宇段
- value length (4B) : 实际消息体的长度。如果为-1，则表示消息为空
- value:消息体。可以为空，比如墓碑(tombstone)消息

v0版本中一个消息的最小长度(RECORD_OVERHEAD_V0)为 crc32 + magic + attributes + key length + value length = 4B + 1B + 1B + 4B + 4B = 14B。
也就是说，v0版本中一条消息的最小长度为14B，如果小于这个值，那么这就是一条破损的消息而不被接收。

&nbsp;

### **4.2.2 v1版本** ###
**(0.10.0 - 0.11.0之前)**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814113839.png)

- crc32(4B):crc32校验值。校验范围为magic至value之间
- magic(1B):消息格式版本号，此版本的magic值为1
- timestamp(8B):时间戳，timestamp类型由broker端参数 og.message.timestamp.type来配置，默认值为CreateTime，即采用生产者创建消息时的时间戳
- attributes(1B):消息的属性。总共占1个字节,低3位表示压缩类型: 0表示NONE、1表示GZIP、2表示SNAPPY、3表示LZ4(LZ4自Kafka0.9.x引入)，第4个位(bit): 0表示timestamp类型为CreateTime, 而1表示timestamp类型为LogAppendTime
- key length(4B) : 表示消息的key的长度。如果为-1，则表示没有设置key，即key = null
- key:可选，如果没有key则无此宇段
- value length (4B) : 实际消息体的长度。如果为寸，则表示消息为空
- value:消息体。可以为空，比如墓碑(tombstone)消息

&nbsp;

### **4.2.3 消息压缩** ###
- Kafka实现的压缩方式是将多条消息一起进行压缩，这样可以保证较好的压缩效果。在一般情况下，生产者发送的压缩数据在broker中也是保持压缩状态进行存储的，消费者从服务端获取的也是压缩的消息，消费者在处理消息之前才会解压消息 ，这样保持了端到端的压缩
- Kafka日志中使用哪种压缩方式是通过参数compression.type来配置的，默认值为“producer”，表示保留生产者使用的压缩方式。这个参数还可以配置为“gzip”、“snappy”、“lz4”，分别对应GZIP、SNAPPY、LZ4这3种压缩算法。如果参数compression.type配置为“uncompressed”，则表示不压缩

**当消息压缩时是将整个消息集进行压缩作为内层消息(inner message)，内层消息整体作为外层(wrapper message)的value，其结构如图所示:**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814143708.png)

**对于未压缩的情形，图右内层消息中最后一条的offset理应是1030，但被压缩之后就变成了5，而这个1030被赋予给了外层的offset**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814144740.png)

&nbsp;

### **4.2.4 v2版本** ###
**(0.11.0以后)**

#### **4.2.4.1 变长字段** ####
Kafka从0.11.0版本开始所使用的消息格式版本为v2，这个版本的消息相比v0和v1的版本而言改动很大，同时还参考了Protocol Buffer而引入了变长整型(Varints)和ZigZag编码。

&nbsp;

#### **4.2.4.2 消息格式** ####
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814145833.png)

>
- 消息格式 Record 的关键字段，可以看到内部字段大量采用了Varints，这样Kafka可以根据具体的值来确定需要几个字节来保存。
- v2版本的消息格式去掉了crc字段(从Record转移到RecordBatch)，另外增加了length(消息总长度〉、timestamp delta(时间戳增量)、offset delta(位移增量)和headers信息，并且attributes字段被弃用了

**Record**

>
- length:消息总长度
- attributes: 弃用，但还是在消息格式中占据1B的大小，以备未来的格式扩展
- timestamp delta: 时间戳增量。通常一个timestamp需要占用8个字节，如果像这里一样保存与RecordBatch的起始时间戳的差值，则可以进一步节省占用的字节数
- offset delta: 位移增量。保存与RecordBatch起始位移的差值，可以节省占用的字节数
- headers:这个字段用来支持应用级别的扩展，而不需要像v0和v1版本一样不得不将一些应用级别的属性值嵌入消息体

**RecordBatch**

>
- first offset:表示当前RecordBatch的起始位移
- length:计算从 partition leader epoeh字段开始到末尾的长度
- partitio leader epoeh:分区leader纪元，可以看作分区leader的版本号或更新次数
- magic:消息格式的版本号，对v2版本而言， magie等于2。
- attributes:消息属性，注意这里占用了两个字节。低3位表示压缩格式，可以参考v0和v1;第4位表示时间戳类型;第5位表示此RecordBatch是否处于事务中，0表示非事务，1表示事务;第6位表示是否是控制消息 (ControlBatch)，0表示非控制消息，而1表示是控制消息，控制消息用来支持事务功能
- lastoffsetdelta: RecordBatch中最后一个Record的offset与first offset的差值。主要被broker用来确保RecordBatch中Record组装的正确性
- first timestamp: RecordBatch中第一条Record的时间戳
- max timestamp: RecordBatch中最大的时间戳，一般情况下是指最后一个Record的时间戳，和last offset delta的作用一样，用来确保消息组装的正确性
- producer id : PID，用来支持幂等和事务
- producer epoeh:和 producer id一样，用来支持幂等和事务
- first sequenee:和 produeer id、 producer epoeh一样，用来支持幂等和事务
- records count: RecordBatch中Record的个数

&nbsp;

## **4.3 日志索引** ##

>
- 每个日志分段文件对应了两个索引文件，主要用来提高查找消息的效率
- 偏移量索引文件用来建立消息偏移量(offset)到物理地址之间的映射关系，方便快速定位消息所在的物理文件位置
- 时间戳索引文件则根据指定的时间戳(timestamp)来查找对应的偏移量信息

Kafka中的索引文件以稀疏索引(sparse index)的方式构造消息的索引，它并不保证每个消息在索引文件中都有对应的索引项。
每当写入一定量(由broker端参数log.index.interval.bytes指定，默认值为4096，即4KB)的消息时，偏移量索引文件和时间戳索引文件分别增加一个偏移量索引项和时间戳索引项，增大或减小log.index.interval.bytes的值，对应地可以增加或缩小索引项的密度。

### **4.3.1 偏移量索引** ###
**偏移量索引项的格式如图所示，每个索引项占用8个字节，分为两个部分：**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814162054.png)

>
1. relativeOffset:相对偏移量，表示消息相对于baseOffset的偏移量，占用4个字节，当前索引文件的文件名即为baseOffset的值
2. position:物理地址，也就是消息在日志分段文件中对应的物理位置，占用4个字节

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814162434.png)

**如果我们要查找偏移量为23的消息，那么应该怎么做呢?**

>
首先通过二分法在偏移量索引文件中找到不大于23的最大索引项，即[22, 656]，然后从日志分段文件中的物理位置656开始顺序查找偏移量为23的消息

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814163024.png)

**如果要查找偏移量为268的消息，那么应该怎么办呢?**

>
首先肯定是定位到baseOffset为251的日志分段，然后计算相对偏移量relativeOffset = 268 - 251 = 17，之后再在对应的索引文件中找到不大于17的索引项，最后根据索引项中的position定位到具体的日志分段文件位置开始查找目标消息

**那么又是如何查找baseOffset为251的日志分段的呢?**

>
这里并不是顺序查找，而是用了跳跃表的结构。Kafka的每个日志对象中使用了ConcurrentSkipListMap来保存各个日志分段，每个日志分段的baseOffset作为key，这样可以根据指定偏移量来快速定位到消息所在的日志分段

> **Kafka强制要求索引文件大小必须是索引项大小的整数倍,对偏移量索引文件而言，必须为8的整数倍**

&nbsp;

### **4.3.2 时间戳索引** ###
**时间戳索引项的格式如图所示，每个索引占用12个字节，分为两个部分：**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814163513.png)

>
1. timestamp: 当前日志分段最大的时间戳
2. relativeOffset:时间戳所对应的消息的相对偏移量

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814163633.png)

>
1. 将targetTimeStamp和每个日志分段中的最大时间戳largestTimeStamp逐一对比，直到找到不小于targetTimeStamp的largestTimeStamp所对应的日志分段。日志分段中的largestTimeStamp的计算是先查询该日志分段所对应的时间戳索引文件，找到最后一条索引项，若最后一条索引项的时间戳字段值大于0，则取其值，否则取该日志分段的最近修改时间
2. 找到相应的日志分段之后，在时间戳索引文件中使用二分查找算法查找到不大于targetTimeStamp的最大索引项，即[1526384718283, 28]，如此便找到了 一个相对偏移量28
3. 在偏移量索引文件中使用二分算法查找到不大于28的最大索引项，即[26, 838]
4. 从步骤1中找到日志分段文件中的838的物理位置开始查找不小于targetTimeStamp的消息

&nbsp;

## **4.4 日志清理** ##
### **4.4.1 日志删除** ###
**1.基于时间（默认7天过期）：日志删除任务会检查当前日志文件中是否有保留时间超过设定的阈值（retentionMs）找寻找可删除的日志分段文件集合(deletableSegments)**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814165125.png)

>
1. 查找过期的日志分段文件，根据日志分段中最大的时间戳 largestTimeStamp 来计算的；
2. 须要保证有一个活跃的日志分段activeSegment；
3. 日志分段文件添加上“.deleted”的后缀，最后由定时任务执行删除，默认1分钟执行一次

**2.基于日志大小：当前日志的大小是否超过设定的阈值(retentionSize)**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814165204.png)

**3.基于日志起始偏移量：小于logStartOffset可以删除**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814165221.png)

>
一般情况下，日志文件的起始偏移量 logStartOffset 等于第一个日志分段的 baseOffset，但这并不是绝对的
基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段。

&nbsp;

### **4.4.2 日志压缩** ###

> Log Compaction对于有相同key的不同value值，只保留最后一个版本。如果应用只关心key对应的最新value值，则可以开启Kafka的日志清理功能，Kafka会定期将相同key的消息进行合井，只保留最新的value值

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814165512.png)

&nbsp;

## **4.5 磁盘存储** ##
### **4.5.1 页缓存** ###
**页缓存是操作系统实现的一种主要的磁盘缓存，以此用来减少对磁盘 I/O 的操作**

>
- 当一个进程准备读取磁盘上的文件内容时，操作系统会先查看待读取的数据所在的页(page)是否在页缓存(pagecache)中，如果存在(命中)则直接返回数据，从而避免了对物理磁盘的 I/O 操作；如果没有命中，则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存，之后再将数据返回给进程。
- 如果一个进程需要将数据写入磁盘，那么操作系统也会检测数据对应的页是否在页缓存中，如果不存在，则会先在页缓存中添加相应的页，最后将数据写入对应的页。被修改过后的页也就变成了脏页，操作系统会在合适的时间把脏页中的数据写入磁盘，以保持数据的一致性 。

> **Kafka中大量使用了页缓存，这是Kafka实现高吞吐的重要因素之一**

&nbsp;

### **4.5.2 零拷贝** ###

> **Kafka还使用零拷贝(Zero-Copy)技术来进一步提升性能**

>
- 所谓的零拷贝是指将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手
- 零拷贝大大提高了应用程序的性能，减少了内核和用户模式之间的上下文切换
- 对 Linux 操作系统而言，零拷贝技术依赖于底层的sendfile()方法实现；对应于 Java 语言，FileChannal.transferTo()方法的底层实现就是sendfile()方法

**非零拷贝技术如图：**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814170651.png)

>
1. 调用read()时，文件A中的内容被复制到了内核模式下的ReadBuffer中
2. CPU控制将内核模式数据复制到用户模式下
3. 调用write()时，将用户模式下的内容复制到内核模式下的Socket Buffer中
4. 将内核模式下的SocketBuffer的数据复制到网卡设备中传送

> **数据复制4次，内核和用户模式的上下文的切换4次**

**零拷贝技术如如：**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190814171424.png)

>
- 零拷贝技术通过DMA(Direct Memory Access)技术将文件内容复制到内核模式下的Read Buffer中
- 没有数据被复制到Socket Buffer，只有包含数据的位置和长度的信息的文件描述符被加到Socket Buffer中
- DMA引擎直接将数据从内核模式中传递到网卡设备(协议引擎)
- 数据复制2次，内核和用户模式的上下文切换2次
- 零拷贝是针对内核模式而言的，数据在内核模式下实现了零拷贝
