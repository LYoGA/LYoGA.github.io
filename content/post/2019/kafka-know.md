---
title: "Kafka Know"
date: 2019-10-08T20:09:34+08:00
lastmod: 2019-10-08T20:09:34+08:00
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
## **1.Kafka中的ISR、AR又代表什么？ISR的伸缩又指什么** ##

- AR(Assigned Replicas)：分区中的所有副本统称为AR
- ISR(In-Sync Replicas)：所有与leader副本保持一定程度同步的副本(包括leader副本在内)组成ISR
- OSR(Out-of-Sync Replicas)：与leader副本同步滞后过多的副本(不包括leader副本)组成OSR

**AR = ISR + OSR**


&nbsp;


## **2.ISR伸缩**##
Kafka在启动的时候会开启两个与ISR相关的定时任务，名称分别为"isr-expiration”和 “isr-change-propagation”

**缩减：**

- isr-expiration任务会周期性地检测每个分区是否需要缩减其ISR集合，但检测到ISR集合中有失效副本时，就会收缩ISR。如果某个分区的ISR结合发生变更，则将变更后的数据记录到Zookeeper对应的节点，并且还会将变更记录缓存到isrChangeSet中
- isr-change-propagation任务会周期性(固定值为 2500ms)地检查isrChangeSet，如果发现isrChangeSet中有ISR集合的变更记录，会将isrChangeSet中的信息保存到Zookeeper的节点中，并且Kafka控制器会为此添加一个Watcher，当这个节点发生数据变化，当这个节点中有子节点发生变化时会触发Watcher的动作，以此通知控制器更新相关元数据信息井向它管理的broker节点发送更新元数据的请求，最后删除/isr_change_ notification路径下已经处理过的节点

**扩充：**

- 追赶上leader副本的判定准则是此副本的LEO是否不小于leader副本的HW，ISR扩充之后同样会更新ZooKeeper中的/brokers/topics/(topic)/partition/(parititon)/state节点和isrChangeSet，之后的步骤就和ISR收缩时的相同。

&nbsp;

## **3.Kafka中的HW、LEO、LSO、LW等分别代表什么？**##

- HW(High Watermark)：俗称高水位，它标识了一个特定的消息偏移量(offset)，消费者只能拉取到这个offset之前的消息
- LEO(Log End Offset)：标识当前日志文件中下一条待写入消息的offset
- LSO(Last Stable offset)：Last Stable Offset对未完成的事务而言，LSO的值等于事务中第一条消息的位置(firstUnstableOffset)，对已完成的事务而言，它的值同HW相同
- LW(Low Watermark)：低水位, 代表AR集合中最小的logStartOffset值

&nbsp;

## **4.Kafka中是怎么体现消息顺序性的？**##
kafka每个partition中的消息在写入时都是有序的（顺序写磁盘），消费时，每个partition只能被每一个group中的一个消费者消费，保证了消费时也是有序的。
整个topic不保证有序。如果为了保证topic整个有序，那么将partition调整为1.

&nbsp;


## **5.Kafka中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？**##
- 序列化器（必备）：生产者需要用序列化器(Serializer)把对象转换成字节数组才能通过网络发送给Kafka。而在对侧，消费者需要用反序列化器(Deserializer)把从Kafka中收到的字节数组转换成相应的对象。
- 分区器（必备）：消息经过序列化之后就需要确定它发往的分区，如果消息ProducerRecord中指定了partition字段，发往指定分区；如果消息ProducerRecord中没有指定partition字段，那么就需要依赖分区器，根据key这个字段来计算partition的值。key不为null时，计算所得分区为所有分区之一；如果为key为null，计算所得分区为所有可用分区中的一个
- 拦截器（可没有）：过滤某些消息

**顺序：拦截器->序列化器->分区器**

&nbsp;

## **6.Kafka生产者客户端的整体结构是什么样子的？**##
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191020154246.png)

&nbsp;

## **7.Kafka生产者客户端中使用了几个线程来处理？分别是什么？**##
- 整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和Sender线程(发送线程)。在主线程中由KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器(RecordAccumulator，也称为消息收集器)中。Sender线程负责从RecordAccumulator中获取消息并将其发送到Kafka中
- RecordAccumulator主要用来缓存消息以便Sender线程可以批量发送，进而减少网络传输的资源消耗以提升性能(buffer.Memory = 32MB)
- RecordAccumulator中ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches存储消息的数据结构，尾写头读
- ProducerBatch由多个ProducerRecord组成，也就是消息批次，减少网络请求的次数以提升整体的吞吐量
- RecordAccumulator的内部还有一个BufferPool，它主要用来实现ByteBuffer的复用，以实现缓存的高效利用，不过BufferPool只针对特定大小的ByteBuffer进行管理，而其他大小的ByteBuffer不会缓存进BufferPool（batch.size = 16KB）
- Sender从 ecordAccumulator中获取缓存的消息之后，将原本<分区，Deque<ProducerBatch>>转变成<Node（broker节点）, List<ProducerBatch>的形式
- 在转换成<Node, List<ProducerBatch>>的形式之后，Sender进一步封装成<Node, Request（协议>的形式，这样就可以将Request请求发往各个Node
- 请求在从Sender线程发往Kafka之前还会保存到InFlightRequests，Map<String（NodeId）, Deque<NetworkClient.InFlightRequest>> requests，它的主要作用是缓存了已经发出去但还没有收到响应的请求，maxInFlightRequestsPerConnection = 5（默认最大连接数）
- maxInFlightRequestsPerConnection <= 5的原因，Server端的ProducerStateManager实例会缓存每个PID在每个Topic-Partition上发送的最近5个batch数据（这个5是写死的，至于为什么是5，可能跟经验有关，当不设置幂等性时，当这个设置为5时，性能相对来说较高），如果超过5，ProducerStateManager就会将最旧的batch数据清除

&nbsp;

## **8.Kafka的旧版Scala的消费者客户端的设计有什么缺陷？**##
- 旧版的消费者客户端是使用ZooKeeper的监听器(Watcher）来实现消费者之间的协调，消费者协调器和组协调器的概念是针对新版的消费者客户端而言的
- 消费位移之前也是存储在Zookeeper上面的，后来改为内部主题__consumer_offsets
- 要种依赖Zookeeper会有两个问题：1.羊群效应(HerdEffect)：所谓的羊群效应是指ZooKeeper中一个被监听的节点变化，大量的Watcher通知被发送到客户端，导致在通知期间的其他操作延迟，也有可能发生类似死锁的情况；2.脑裂问题(Split Brain)：消费者进行再均衡操作时每个消费者都与ZooKeeper进行通信以判断消费者或broker变化的情况，由于ZooKeeper本身的特性，可能导致在同一时刻各个消费者获取的状态不一致，这样会导致异常问题发生

&nbsp;

## **9.“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？如果不正确，那么有没有什么hack的手段？**##
正确。同一个分区只能被到同一个消费组中的一个消费者。

&nbsp;

## **10.消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1?**##
offset+1，表示下一条需要拉取的消息的位置
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191020164144.png)

&nbsp;

## **11.有哪些情形会造成重复消费&消息丢失？**##
对于offset的commit，Kafka Consumer Java Client支持两种模式：由KafkaConsumer自动提交，或者是用户通过调用commitSync、commitAsync方法的方式完成offset的提交。(对于用户而言，位移提交分为同步提交和异步提交，从consumer角度而言，位移分为同步提交和异步提交)

**自动提交：**

- Kafka中默认的消费位移的提交方式是自动提交，enable.auto.commit = true，周期是定时提交，auto.commit.interval.ms = 5s，自动提交的动作是在poll()方法的逻辑中完成的，会在每次拉取请求之间检查是否可以进行位移提交
- 自动提交消息丢失：若poll()返回消息后是异步处理，或同步处理耗时过长，则自动提交时，若消息没有处理完，也会将它们提交，在Kafka broker看来，这些消息已经完成处理自动提交时，此时consumer崩溃的话，未处理的消息就会丢失
- 自动提交消息重复：Consumer每5秒提交一次位移，若提交位移后3秒发生Rebalance，所有Consumer从上次提交的位移处继续消费

**手动提交：同步提交&异步提交**

```
// 同步提交
properties.put("enable.auto.commit", "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
        try {
           consumer.commitSync();
        } catch (Exception e) {
            System.out.println("commit failed");
        }

    }
}
```

- 提交KafkaConsumer#poll()返回的最新位移
- 同步操作，一直等待，直到位移被成功提交才会返回
- 需要处理完poll方法返回的所有消息后，才提交位移，否则会出现消息丢失
- Consumer处于阻塞状态，直到远端的Broker返回提交结果，才会结束
- 因为应用程序而非资源限制而导致的阻塞都可能是系统的瓶颈，会影响整个应用程序的TPS

```
// 异步提交
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    process(records); // 处理消息
    consumer.commitAsync((offsets, exception) -> {
        if (exception != null)
            handle(exception);
    });
}
```

- 异步操作，立即返回，不会阻塞，不会影响Consumer应用的TPS，Kafka也提供了回调函数
- commitAsync不能代替commitSync，因为commitAsync不会自动重试
- 如果异步提交后再重试，提交的位移值很可能已经过期，因此异步提交的重试是没有意义的

```
// 同步提交 + 异步提交
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
        process(records); // 处理消息
        commitAysnc(); // 使用异步提交规避阻塞
    }
} catch (Exception e) {
    handle(e); // 处理异常
} finally {
    try {
        consumer.commitSync(); // 最后一次（异常/应用关闭）提交使用同步阻塞式提交
    } finally {
        consumer.close();
    }
}
```

- 手动提交需要组合commitSync和commitAsync，达到最优效果
- 利用commitSync的自动重试来规避瞬时错误，如网络瞬时抖动、Broker端的GC等
- 利用commitAsync的非阻塞性，保证Consumer应用的TPS

```
// 精细化提交
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
int count = 0;
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
        process(record);  // 处理消息
        // 消费位移是下一条消息的位移，所以+1
        offsets.put(new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1));
        if (count % 100 == 0) {
            // 精细化提交
            consumer.commitAsync(offsets, null); // 回调处理逻辑是 null
        }
        count++;
    }
}
```

- 上面的位移提交方式，都是提交poll方法返回的所有消息的位移，即提交最新一条消息的位移
- 精细化提交
  - commitSync(Map<TopicPartition, OffsetAndMetadata> offsets)
  - commitAsync(Map<TopicPartition, OffsetAndMetadata> offsets, OffsetCommitCallback callback)

&nbsp;

## **12.Kafka JAVA Consumer为什么采用单线程设计**##
- 我们说KafkaConsumer是单线程的设计，严格来说这是不准确的。因为从Kafka 0.10.1.0版本开始，KafkaConsumer就变成双线程的设计，即用户主线程和心跳线程。
- 所谓用户主线程，就是你启动Consumer应用程序main方法的那个线程，而新引入的心跳线程（Heartbeat Thread）只负责定期给对应的Broker机器发送心跳请求，已表示消费者应用的存活性；引入心跳线程还有另一个目的是期望它能将心跳频率与主线程调用KafkaConsumer.poll方法的频率分开，从而解耦真实的消息处理逻辑与消费者组成员存活性管理
- 新版本Consumer设计了单线程+轮询的机智，这种设计能够较好地实现非阻塞式的消息获取，除此之外，单线程的设计能简化Consumer端的设计；Consumer获取到消息后，处理消息的逻辑是否采用多线程，完全由你决定。

&nbsp;

## **13.KafkaConsumer是非线程安全的，那么怎么样实现多线程消费？**##

- 多个线程中无法共享同一个KafkaConsumer实例，会抛出异常（acquire方法用来检验当前是否只有一个线程在操作）；除了wakeup()例外，可以在其他线程中安全地调用来唤醒Consumer
- acquire方法不会造成阻塞等待，可以看成轻量级锁，通过线程操作计数标记的方式来检测线程是否发生并发操作，对应release方法

**方案一：多线程+多KafkaConsumer实例**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191021144423.png)

**优势：**

- 一个线程对应一个KafkaConsumer实例
- 多个线程之间无任何交互，省去保障线程安全方面的开销
- 主题中的每个分区都能只保证只被一个线程处理，容易实现分区内的消息消费顺序性

**不足：**

- 每个线程维护自己的KafkaConsumer实例，必然占用更多系统资源，比如内存、TCP连接等，在资源紧张的系统环境中，方案1劣势会更加明显
- 该方案能使用的线程数受限于Consumer订阅主题的总分区数
- 每个线程完整地执行消息获取和消息处理逻辑，一旦消息逻辑很重，容易造成消费速度过慢，从而引发Rebalance

**方案二：单线程+单KafkaConsumer实例+处理消息Worker线程池**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191021145716.png)

**优势：**

- 将任务切分成了获取消息和消息处理两个部分，分别由不同线程处理
- 高伸缩性，可以独立调节消息获取的线程数以及消息处理的线程数；还可以减少TCP连接对系统资源的消耗

**不足：**

- 无法保证分区内的消费顺序，如果在意消息的先后顺序，方案2的这个劣势是致命的
- 引入多线程组，使得整个消息消费链路被拉长，正确位移提交将会变得异常困难，结果可能出现重复消费问题，如果在意这一点，不推荐使用这个方案

**解决方案：基于滑动窗口实现**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191021150459.png)

- 方案二所呈现的结构通过消费者拉取分批次的消息，然后提交给多线程进行处理，而这里的滑动窗口式的实现方式是将拉取到的消息暂存起来，多个消费线程可以拉取暂存的消息，这个用于暂存消息的缓存大小即为滑动窗口的大小，总体上而言没有太多的变化，不同的是对于消费位移的把控
- 每一个方格代表一个批次的消息，一个滑动窗口包含若干方格，startOffset标注的是当前滑动窗口的起始位置，endOffset标注的是末尾位置。每当startOffset指向的方格中的消息被消费完成，就可以提交这部分的位移，与此同时，窗口向前滑动一格，删除原来startOffset所指方格中对应的消息，并且拉取新的消息进入窗口。滑动窗口的大小固定，所对应的用来暂存消息的缓存大小也就固定了，这部分内存开销可控。

&nbsp;

## **14.topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？**##
## **15.topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？**##
**目前Kafka只支持增加分区数而不支持减少分区数**

**为什么不支持减少分区?**

- 按照Kafka现有的代码逻辑，此功能完全可以实现，不过也会使代码的复杂皮急剧增大。实现此功能需要考虑的因素很多，比如删除的分区中的消息该如何处理?如果随着分区一起消失则消息的可靠性得不到保障
- 如果需要保留则又需要考虑如何保留。直接存储到现有分区的尾部，消息的时间戳就不会递增，如此对于 Spark、 Flink 这类需要消息时间戳(事件时间)的组件将会受到影响
- 如果分散插入现有的分区，那么在消息量很大的时候，内部的数据复制会占用很大的资源，而且在复制期间，此主题的可用性又如何得到保障?
- 与此同时，顺序性问题、事务性问题，以及分区和副本的状态机切换问题都是不得不面对的
- 反观这个功能的收益点却是很低的，如采真的需要实现此类功能，则完全可以重新创建一个分区数较小的主题，然后将现有主题中的消息按照既定的逻辑复制过去即可

&nbsp;

## **16.创建topic时如何选择合适的分区数？**##
从吞吐量方面考虑，增加合适的分区数可以在一定程度上提升整体吞吐量，但超过对应的阈值之后吞吐量不升反降。
如果应用对吞吐量有一定程度上的要求，则建议在投入生产环境之前对同款硬件资源做一个完备的吞吐量相关的测试，以找到合适的分区数阈值区间

有些应用场景会要求主题中的消息都能保证顺序性，这种情况下在创建主题时可以设定分区数为1，通过分区有序性的这一特性来达到主题有序性的目的

一般情况下，根据预估的吞吐量及是否与 key相关的规则来设定分区数即可，后期可以通过增加分区数、增加broker或分区重分配等手段来进行改进

&nbsp;

## **17.Kafka目前有那些内部topic，它们都有什么特征？各自的作用又是什么？**##
内部topic__consumer_offsets：消费者位移提交的内容最终会保存在该内部主题
协调者中保存了消费组元数据，包括了消费者提交的偏移量和消费组分配的状态数组，这两个数组都是以内部主题__consumer_offsets的形式存储在服务端的协调者节点上。消费组元数据是按topic的形式存储，默认情况下__consumer_offsets有50个分区

内部topic__transaction_state：TransactionCoordinator会将事务状态持久化到该主题，并且用于TransactionCoordinator宕机之后的恢复

&nbsp;

## **18.优先副本是什么？它有什么特殊的作用？**##
为了能够有效地治理负载失衡的情况，Kafka引入了优先副本(preferred replica)的概念。
所谓的优先副本是指在AR集合列表中的第一个副本。比如上面主题topic-partitions中分区0的AR集合列表(Repl工cas)为[1,2,0]，那么分区0的优先副本即为1。理想情况下，优先副本就是该分区的leader副本，所以也可以称之为preferredleader。Kafka要确保所有主题的优先副本在Kafka集群中均匀分布，这样就保证了所有分区的leader均衡分布。如果leader分布过于集中，就会造成集群负载不均衡。
所谓的优先副本的选举是指通过一定的方式促使优先副本选举为leader副本，以此来促进集群的负载均衡，这一行为也可以称为“分区平衡” 。

Kafka提供分区自动平衡功能，定时任务（默认300s）轮询所有broker节点，计算每个broker节点的分区不平衡率(broker中的不平衡率=非优先副本的leader个数/分区总数)是否超过参数默认值为10%，如果超过设定比值自动执行优先副本的选举动作以求分区平衡

生产环境不建议开启此功能，因为可能会引起负面性能问题，最好通过监控手段，然后手动执行分区平衡

&nbsp;

## **19.Kafka有哪几处地方有分区分配的概念？简述大致的过程及原理**##
**1.创建topic，分区分配至broker**

1. 根据broker的长度随机生成一个有效brokerId，作为startIndex分配给第一个partitions
2. 根据broker的长度随机生成指定副本的间隔，根据间隔生成该分区的所有副本
3. 轮询所有分区，每完成一个分区副本分配，currentPartitionId += 1 继续为下一个分区分配副本
```
private def assignReplicasToBrokersRackUnaware(nPartitions: Int, // 分区数
                                               replicationFactor: Int, // 副本因子
                                               brokerList: Seq[Int], // 集群中broker列表
                                               fixedStartIndex: Int, //起始索引，即第 一个副本分自己的位置，默认值为-1
                                               startPartitionId: Int): //起始分区编号，默认值为-1
Map[Int, Seq[Int]] = {
  val ret = mutable.Map[Int, Seq[Int]]() //保存分配结果的集合
  val brokerArray = brokerList.toArray //brokerId的列表
  //如果起始索引fixedStartIndex小于0，则根据broker列表长度随机生成一个，以此来保证是有效的brokerId
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  //确保起始分区号不小于0
  var currentPartitionId = math.max(0, startPartitionId)
  //指定了副本的间隔，目的是为了更均匀地将副本分配到不同的broker上
  var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  //轮询所有分区，将每个分区的副本分配到不同的broker上
  for (_ <- 0 until nPartitions) {
    //只有在所有遍历过Broker数目个分区后才将位移加一
    if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0))
      nextReplicaShift += 1
    //当前分区id加上起始位置，对Brokersize取余得到第一个副本的位置
    val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
    val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
    //保存该分区所有副本分自己的broker集合
    for (j <- 0 until replicationFactor - 1)
      //计算出每个副本的位置 计算方法是replicaIndex：
      //val shift = 1 + (nextReplicaShift + j) % ( brokerList.size - 1)
      //(firstReplicaIndex + shift) %  brokerList.size
      replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
    //保存该分区所有副本的分配信息
    ret.put(currentPartitionId, replicaBuffer)
    //继续为 下一个分区分配副本
    currentPartitionId += 1
  }
  ret
}
```

**2.消费者与订阅主题之间的分区分配策略**

1. RangeAssignor分配策略（区间分配）
假设n = 分区数 / 消费者数量， m = 分区数 % 消费者数量，那么前m个消费者每个分配n + 1个分区，后面的(消费者数量-m)个消费者每个分配 n个分区。
分区=3，消费者2，分配不均  

2. RoundRobinAssignor分配策略（轮询分配）
同一个消费组内的消费者订阅的信息都是相同的时候，轮询可以做到分配均匀，但是如果不同的话，无法做到分配均匀

3. StickyAssignor分配策略（粘性分配）
  - 分区的分配要尽可能均匀
  - 分区的分配尽可能与上次分配的保持相同，减少不必要的分区移动(即一个分区剥离之前的消费者 ，转而分配给另一个新的消费者)

4. 自定义分区分配策略

&nbsp;

## **20.简述Kafka的日志目录结构**##

&nbsp;

## **21.Kafka中有那些索引文件？**##

&nbsp;

## **22.如果我指定了一个offset，Kafka怎么查找到对应的消息？**##

&nbsp;

## **23.如果我指定了一个timestamp，Kafka怎么查找到对应的消息？**##

&nbsp;

## **24.聊一聊你对Kafka的Log Retention的理解**##

&nbsp;

## **25.聊一聊你对Kafka的Log Compaction的理解**##

&nbsp;

## **26.聊一聊你对Kafka底层存储的理解（页缓存、内核层、块层、设备层）**##

&nbsp;

## **30.聊一聊Kafka的延时操作的原理**##

1. 延时生产
如果客户端设置的acks参数不为-1，或者没有成功的消息写入，那么就直接返回结果给客户端，否则就需要创建延时生产操作并存入延时操作管理器，最终要么由外部事件触发，要么由超时触发而执行。
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191010171031.png)

2. 延时拉取
Kafka在处理拉取请求时，会先读取一次日志文件，如果收集不到足够多(fetchMinBytes，由参数fetch.min.bytes配置，默认值为1)的消息，那么就会创建一个延时拉取操作(DelayedFetch)以等待拉取到足够数量 的消息。当延时拉取操作执行时，会再读取一次日志文件，然后将拉取结果返回给follower副本。
延时拉取操作同样是由超时触发或外部事件触发而被执行的。 超时触发很好理解，就是等到超时时间之后触发第二次读取日志文件的操作。外部事件触发就稍复杂了一些，因为拉取请求不单单由follower副本发起，也可以由消费者客户端发起，两种情况所对应的外部事件也是不同的。如果是follower副本的延时拉取，它的外部事件就是消息追加到了leader副本的本地日志文件中；如果是消费者客户端的延时拉取，它的外部事件可以简单地理解为HW的增长。

&nbsp;

## **31.聊一聊Kafka控制器的作用**##
作用：
1. 在Kafka集群中会有一个或多个broker，其中有一个broker会被选举为控制器(Kafka Controller)，它负责管理整个集群中所有分区和副本的状态
2. 当某个分区的leader副本出现故障时，由控制器负责为该分区选举新的leader副本
3. 当检测到某个分区的ISR集合发生变化时，由控制器负责通知所有broker更新其元数据信息
4. 当使用kafka-topics.sh脚本为某个topic增加分区数量时，同样还是由控制器负责分区的重新分配

&nbsp;

## **32.消费再均衡的原理是什么？（提示：消费者协调器和消费组协调器）**##
旧版客户端是通过使用zookeeper的监听器去实现再均衡，但可能出现羊群效应和脑裂问题
新版客户端是通过消费者协调器和消费组协调器来进行再均衡的

**新增消费者再均衡：**

1. FIND_COORDINATOR：消费者需要确定它所属的消费组对应的GroupCoordinator所在的broker，并创建与改broker相互通信的网络连接。如果有则不需要在查找，如果没有存该信息的话，需要向集群中负载最小的节点发送FindCoordinatorRequest来查找
具体查找GroupCoordinator的方式是先根据消费组groupId的哈希值计算_consumer_offsets中的分区编号=>Utils.abs(groupid.hashCode) % consumer_offsets主题分区数量，确定所在分区之后再查找此分区leader副本所在的broker节点，即为这个groupId所对应的GroupCoordinator节点
2. JOIN_GROUP：消费者会向GroupCoordinator发送JoinGroupRequest请求，阻塞等待Kafka服务端的响应。
  - 选举消费组leader
  - 选举分区分配策略
3. SYNC_GROUP(同步信息)
leader消费者根据在第二阶段中选举出来的分区分配策略来实施具体的分区分配，然后将分配结果告知GroupCoordinator，之后各个消费者会向GroupCoordinator发送SyncGroupRequest请求来同步分配方案
4. HEARTBEAT
消费者通过向GroupCoordinator发送心跳来维持它们与消费组的从属关系，以及它们对分区的所有权关系

&nbsp;

## **33.Kafka中的幂等是怎么实现的**##
kafka开启幂等性功能：生产者设置参数enable.idempotence=true（retries > 0 && acks = -1）
实现原理：为了实现生产者的幂等性，kafka引入producer id（PID）和序列号（sequence number）。对于每个PID，消息发送到的每一个分区都有对应的序列号，这些序列号从0开始单调递增。生产者每发送一条消息就会将<PID，分区>对应的序列号的值加1。broker端会在内存中为每一对<PID，分区>维护一个序列号。对于收到的每一条消息，只有当它的序列号的值比broker端中维护的对应的序列号的值大1(即SN_new=SN_old+1)时，broker才会接收它。如果SN_new<SN_old+1，那么说明消息被重复写入，broker可以直接将其丢弃。如果 SN_new> SN_old + 1，那么说明中间有数据尚未写入，出现了乱序，暗示可能有消息丢失，对应的生产者会抛出OutOfOrderSequenceException。
这里的PID是全局唯一的，Producer故障后重新启动后会被分配一个新的PID，这也是幂等性无法做到跨会话的一个原因。
引入序列号来实现幕等也只是针对每一对<PID， 分区>而言的，也就是说，**Kafka的幂等只能保证单个生产者会话(session)中单分区的事等**

Producer-PID申请?
Client通过向Server发送一个InitProducerIdRequest请求获取PID幂等性时，是选择一台连接数最少的Broker发送这个请求），
这里实际上是调用了TransactionCoordinator（Broker在启动server服务时都会初始化这个实例的handleInitProducerId()，而这个方法内部实际上是通过ProducerIdManager的generateProducerId()方法产生一个PID。
ProducerIdManager是在TransactionCoordinator对象初始化时初始化的，这个对象主要是用来管理PID信息：
- 在本地的 PID 端用完了或者处于新建状态时，申请PID段（默认情况下，每次申请1000个PID）；
- TransactionCoordinator对象通过generateProducerId()方法获取下一个可以使用的PID；

PID端申请是向ZooKeeper申请，zk中有一个/latest_producer_id_block节点，每个Broker向zk申请一个PID段后，都会把自己申请的PID段信息写入到这个节点，这样当其他Broker再申请PID段时，会首先读写这个节点的信息，然后根据block_end选择一个PID段，最后再把信息写会到zk的这个节点，这个节点信息格式如下所示：
```
{"version":1,"broker":35,"block_start":"4000","block_end":"4999"}
原理：类似CAS写IP端，版本用于区分
```

简单来说，其实现机制概括为：
- Server端验证batch的sequence number值，不连续时，直接返回异常；
- Client端请求重试时，batch在reenqueue（重试队列）时会根据sequence number值放到合适的位置（有序保证之一）；
- Sender线程发送时，在遍历queue中的batch时，会检查这个batch是否是重试的batch，如果是的话，只有这个batch是最旧的那个需要重试的batch，才允许发送，否则本次发送跳过这个Topic-Partition数据的发送等待下次发送

&nbsp;

## **34.Kafka中的事务是怎么实现的**##
事务要求生产者开启幂等特性。为了实现事务，应用程序必须提供唯一的transactionalId，这个transactionalId通过客户端参数transactional.id来显式设置，而PID是由Kafka内部分配的。
transactionalId与PID一一对应，为了保证新的生产者启动后具有相同transactionalId的旧生产者能够立即失效，每个生产者通过transactionalId获取PID的同时，还会获取一个单调递增的producer epoch。如果使用同一个transactionalId开启两个生产者，那么前一个开启的生产者会报错。

消费端事务：isolation.level设置事务隔离级别（读未提交（默认）、读已提交）
ControlBatch：KafkaConsumer可以通过这个控制消息来判断对应的事务是被提交了还是被中止了，然后结合参数isolation.level配置的隔离级别来决定是否将相应的消息返回给消费端应用

**Consume-transform-Produce的流程**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191014154907.png)
1. 查找TransactionCoordinator
Producer向任意一个brokers发送FindCoordinatorRequest请求来获取TransactionCoordinator的地址。
Utils.abs(transactionalIid.hashCode) % transactionTopicPartitionCount确定分区，之后再找出此分区leader副本所在的broker节点
2. 初始化事务initTransaction
Producer发送InitpidRequest给事务协调器，获取一个Pid。InitpidRequest的处理过程是同步阻塞的，一旦该调用正确返回，Producer就可以开始新的事务。TranactionalId通过InitpidRequest发送给TrancitonCorordinator，然后在Tranaciton Log中记录这<TranacionalId, pid>的映射关系。除了返回PID之外，还具有如下功能：
> - 对PID对应的epoch进行递增，这样可以保证同一个app的不同实例对应的PID是一样的，但是epoch是不同的。
> - 回滚之前的Producer未完成的事务（如果有）。

3. 开始事务beginTransaction
执行Producer的beginTransacion()，它的作用是Producer在本地(TransactionManager)记录下这个transaction的状态为开始状态。
**注意：这个操作并没有通知Transaction Coordinator**
4. Consume-transform-produce loop
> 0. 通过Consumtor消费消息，处理业务逻辑
> 1. producer向TransactionCordinantro发送AddPartitionsToTxnRequest
在producer执行send操作时，如果是第一次给<topic, partion>发送数据，此时会向TrasactionCorrdinator发送一个AddPartitionsToTxnRequest请求，TransactionCorrdinator会在transaction log（transaction_state主题）中记录下tranasactionId和<topic, partion>一个映射关系，并将状态改为begin。
> 2. producer.send发送ProduceRequst
生产者发送数据，虽然没有还没有执行commit或者absrot，但是此时消息已经保存到kafka上，，而且即使后面执行abort，消息也不会删除，只是更改状态字段标识消息为abort状态。
> 3. AddOffsetsToTxnRequest
Producer在调用sendOffsetsToTransaction()方法时，第一步会首先向TransactionCoordinator发送相应的AddOffsetsToTxnRequest请求，TransactionCoordinator在收到这个请求时，会把这个 group.id对应的topic__consumer_offsets的Partition（与写入涉及的 Topic-Partition 一样）保存到事务对应的meta中，之后会持久化相应的事务日志
> 4. TxnOffsetCommitRequest
Producer在收到TransactionCoordinator关于AddOffsetsToTxnRequest请求的结果后，后再次发送TxnOffsetsCommitRequest请求给对应的GroupCoordinator，GroupCoordinator在收到相应的请求后，会将offset信息持久化到consumer offsets log中（包含对应的 PID 信息），但是不会更新到缓存中，除非这个事务commit了，这样的话就可以保证这个offset信息对consumer是不可见的

5. Committing or Aborting a Transaction
在一个事务操作处理完成之后，Producer需要调用commitTransaction()或者abortTransaction()方法来commit或者abort这个事务操作。
无论是Commit还是Abort，对于Producer而言，都是向TransactionCoordinator发送EndTxnRequest请求，这个请求的内容里会标识是commit操作还是abort操作
TransactionCoordinator在收到EndTxnRequest请求后会执行如下操作:
(1)将PREPARE COMMIT或PREPARE ABORT消息写入主题__transaction_state
(2)通过WriteTxnMarkersRequest请求将COMMIT或ABORT信息写入用户所使用的普通主题和__consumer_offsets
(3)将COMPLETE COMMIT或COMPLETE ABORT信息写入内部主题__transaction_state中，到这里，这个事务操作就算真正完成了，TransactionCoordinator缓存的很多关于这个事务的数据可以被清除了。

- http://matt33.com/2018/11/04/kafka-transaction/
- http://www.heartthinkdo.com/?p=2040

&nbsp;

## **35.Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？**##
1. 控制器选举
Kafka中的控制器选举工作依赖于ZooKeeper，成功竞选为控制器的broker会在ZooKeeper中创建/controller这个临时(EPHEMERAL)节点
ZooKeeper中还有一个与控制器有关的/controller_epoch的持久节点，这个节点中存放的是一个整型的controller epoch值。controller epoch用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为“控制器的纪元”。
Kafka通过controller epoch来保证控制器的唯一性，进而保证相关操作的一致性。
控制器在选举成功之后会读取ZooKeeper中各个节点的数据来初始化上下文信息(ControllerContext)，并且需要管理这些上下文信息。比如为某个主题增加了若干分区，控制器在负责创建这些分区的同时要更新上下文信息，并且需要将这些变更信息同步到其他普通的broker节点中。不管是监听器触发的事件，还是定时任务触发的事件，或者是其他事件(比如ControlledShutdown，具体可以参考6.4.2节)都会读取或更新控制器中的上下文信息，那么这样就会涉及多线程间的同步。如果单纯使用锁机制来实现，那么整体的性能会大打折扣。针对这一现象，Kafka的控制器使用单线程基于事件队列的模型，将每个事件都做一层封装，然后按照事件发生的先后顺序暂存 到 LinkedB!ockingQueue 中 ，最后使 用 一个专 用的 线程 (ControllerEventThread)按照 FIFO (FirstInputFirstOutput，先入先出)的原则Jl页序处理各个
事件，这样不需要锁机制就可以在多线程间维护线程安全
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191010175706.png)

2. 分区leader的选举
分区leader副本的选举由控制器负责具体实施(分区创建或者分区原leader下线)
基本思路是按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中。一个分区的AR集合在分配的时候就被指定，并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，而分区的ISR集合中副本的顺序可能会改变。

3. 消费组leader选举
消费组协调器GroupCoordinator会为消费组选举一个消费者作为leader。如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader；如果某一刻leader消费者由于某些原因退出消费组的话，会重新选举，因为消费者信息是以hashMap的形式存储的，所以GroupCoordinator会选取hashMap中第一个键值对的key作为leader，很随意，如同随机一般。

&nbsp;

## **36.失效副本是指什么？有那些应对措施？**##
正常情况下，分区的所有副本都处于ISR集合中，但是难免会有异常情况发生，从而某些副本被剥离出ISR集合中。在ISR集合之外，也就是处于同步失效或功能失效(比如副本处于非存活状态)的副本统称为失效副本，失效副本对应的分区也就称为同步失效分区，即under-replicated分区。

&nbsp;

## **37.多副本下，各个副本中的HW和LEO的演变过程**##

- 生产者客户端发送消息至leader副本中
- 消息被追加到leader副本的本地日志，并且会更新日志的偏移量(LEO)
- follower副本向leader副本请求同步数据
- leader副本所在的服务器读取本地日志，并更新对应拉取的follower副本的信息(Leader副本会记录所有follower副本的LEO)
- leader副本所在的服务器将拉取结果返回给follower副本
- follower副本收到leader副本返回的拉取结果，将消息追加到本地日志中，并更新日志的偏移量信息

>
- follower副本向leader副本拉取消息，在request中会带上自身的LEO信息，leader副本会根据follower副本以及本身的LEO取最小值去更新自身的HW；leader副本返回给follower副本的response中会带上自身的HW
- follower副本各自拉取消息，并更新自身的LEO，于此同时还会取min(自身LEO, Leader副本传来的HW)值来更新自身的HW

异常情况包括数据丢失以及数据不一致的场景，在0.11.0.0之前阶段数据是依据HW进行的，会出现以上异常情况
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191017202644.png)
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191017202735.png)

为了解决以上问题，0.11.0.0开始引入leader epoch概念，在需要截断数据的时候使用leader epoch作为参考依据而不是原本的HW。leader epoch代表leader的纪元信息(epoch)，初始值为 0。每当leader变更 一次，leader epoch的值就会加1，相当于为leader增设了一个版本号。与此同时，每个副本中还会增设一个矢量<LeaderEpoch => StartOffset>，其中StartOffset表示当前LeaderEpoch下写入的第一条消息的偏移量。区别在于follower不是先忙着截断日志而是先发送OffsetsForLeaderEpochRequest请求，判断leaderEpoch是否一致，如果不一致之后的startOffset是什么，用于截断，保证一致性

&nbsp;

## **38.为什么Kafka不支持读写分离？**##
首先从代码层面来说，是完全可以支持的，但从收益点出发的话，不值得，虽然主写从读可以让从节点去分担主节点的负载压力，预防主节点负载过重而从节点却空闲的情况发生，但也有2个明显的缺点：
- 数据一致性问题。数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致。
- 延时问题。Kafka的主从同步需要经历网络→主节点内存→主节点磁盘→网络→从节点内存→从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。
Kafka只支持主写主读有几个优点：可以简化代码的实现逻辑，减少出错的可能；将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而且对用户可控;没有延时的影响；在副本稳定的情况下，不会出现数据不一致的情况。

&nbsp;

## **39.Kafka在可靠性方面做了哪些改进？（HW, LeaderEpoch）**##
- 从数据存储方面来说，多副本机制，采用ISR机制来保证可靠性，配合min.insync.replicas(最小副本数，防止只剩leader副本)以及unclean.leader.election.enable=false确保不会从非ISR中选举leader副本，并且在follower副本同步leader副本数据的时候，依靠leaderEpoch来解决数据丢失和不一致性问题
- 从生产者发送消息来说，ack=-1确保leader副本以及ISR中的所有follower副本都写入该条数据
- 从消费者消费消息来说，在执行手动位移提交的时候也要遵循一个原则：如果消息没有被成功消费，那么就不能提交所对应的消费位移。对于高可靠要求的应用来说，宁愿重复消费也不应该因为消费异常而导致消息丢失。如果由于应用异常，导致部分消息消费失败，为了不影响整体消费进度，可以将此类消息存入死信队列中

&nbsp;

## **40.Kafka中怎么实现死信队列和重试队列？**##
死信队列：由于某些原因消息无法被正确地投递，为了确保消息不会被无故地丢弃，一般将其置于一个特殊角色的队列，这个队列一般称为死信队列
重试队列：重试队列其实可以看作一种回退队列，具体指消费端消费消息失败时，为了防止消息无故丢失而重新将消息回该到broker中。与回退队列不同的是，重试队列一般分成多个重试等级，每个重试等级一般也会设置重新投递延时，重试次数越多投递延时就越大。

&nbsp;

## **41.Kafka中的延迟队列怎么实现（这题被问的比事务那题还要多！！！听说你会Kafka，那你说说延迟队列怎么实现？）**##
单层时间轮

&nbsp;

## **42.Kafka中怎么做消息轨迹？**##
内部主题trace_topic存储消费轨迹

&nbsp;

## **43.Kafka有哪些指标需要着重关注？**##
生产者：每分钟生产消息数目、消息发送延时情况
消费者：消息积压数目、Partition消息积压分布、每分钟消费消息数目、消费延时情况

怎么计算Lag？(注意read_uncommitted和read_committed状态下的不同)

&nbsp;

## **44.Kafka的那些设计让它有如此高的性能？**##
1.从宏观架构层面
- 利用Partition实行并行处理，水平扩展能力
- ISR实现可用性与数据一致性的动态平衡

2.具体实现层面
- 顺序写磁盘
- Kafka大量使用页缓存（Page Cache），充分利用所有空闲内存（非JVM内存，减少GC负担，并且如果进程重启，JVM内的Cache会失效，但Page Cache依旧可用）
- sendfile技术（零拷贝），从数据复制4次，线程上下文切换4次，变成数据复制2次，线程上下文切换2次
- 批量处理消息减少网络开销，提高写磁盘效率；支持数据压缩降低网络负载；高效序列化方式

&nbsp;

## **45.怎么样才能确保Kafka极大程度上的可靠性？**##
高可靠性存储分析
- 副本机制
- ISR
- leader选举
- ack=all

高可靠性使用分析
- 消息传输保障
- 高可靠性配置，副本数最少3，2<=min.insync.replicas<=replication.factor
- request.required.acks=-1(all)，producer.type=sync

&nbsp;

## **46.kafka与rocket mq的区别##
http://jm.taobao.org/2016/03/24/rmq-vs-kafka/
