---
title: "Redis 总结"
date: 2019-12-22T15:55:19+08:00
lastmod: 2019-12-22T15:55:19+08:00
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
# **一、基本数据结构**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191228154526.png)

&nbsp;

## **1.1 字符串对象**

- **字符串是redis使用最多的数据结构，除了本身作为SET/GET的操作对象外，数据库中的所有key，以及执行命令时提供的参数，都是用字符串作为载体的**
- **当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M**

**底层两种实现：**

- REDIS_ENCODING_INT使用long类型保存long的值
- REDIS_ENCODING_ROW使用sdshdr保存sds、long long、double、long double等

```
// 简单动态字符串
struct sdshdr {
    // 记录buf数组中已使用字节的数量，等于记录SDS所保存字符串的长度
    int len;
    // 记录buf数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
}
```

- redis的字符串表示为sds，而不是C字符串（以\0结尾的char*）（C字符串只会作为字符串面量用在一些无须对字符串值进行修改的地方，比如打印日志）
- 对比C字符串，sds有以下特性
  - 常数复杂度O(1)获取字符串长度
  - 杜绝缓冲区溢出
  - 减少修改字符串长度时所需的内存重分配次数（空间预分配&惰性空间释放）
  - 二进制安全
  - 兼容部分C字符串函数

**为什么对于字符串造轮子？**

- 在redis的各种操作中，都会频繁使用字符串的长度和append操作，对于char[]来说，长度操作是O(N)的，append会引起N次内存重分配。
- redis分为client和server，传输的协议内容必须是二进制安全的，而C的字符串必须保证是‘\0’结尾

&nbsp;

## **1.2 列表对象**
**类似JAVA双向链表**

**底层两种实现：**

- REDIS_ENCODING_ZIPLIST（满足保存所有字符串元素长度都小于64字节&元素小于512个）
- REDIS_ENCODING_LINKEDLIST

```
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
 
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
} list;
```

&nbsp;

## **1.3 哈希对象**
### **底层实现：**

- REDIS_ENCODING_ZIPLIST（满足保存所有字符串元素长度都小于64字节&元素小于512个）
- REDIS_ENCODING_HT(字典)

**当创建新的哈希表时，默认是使用压缩列表作为底层数据结构的，因为省内存，只有当触发了阈值才会转为字典**

```
// 字典
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash索引，当rehash不在进行时，值为-1
    int trehashidx;
  	// 目前正在运行的安全迭代器的数量
    int iterators;
} dict;
 
// 哈希表
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;  
} dictht;
 
// 哈希表节点
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union{
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

- 字典是由键值对构成的抽象数据结构
- redis中的数据库和哈希键值对都基于字典实现
- 字典的底层实现为哈希表，每个字典含有2个哈希表，一般只是用0号哈希表，1号哈希表是在rehash过程中才使用的
  - 因为性能，我们知道，当hash表出现太多碰撞的话，查找会由O(1)增加到O(MAXLEN)，通过rehash，从而将碰撞率降低，当hash表大小和节点数量维持在1：1时候性能最优，就是O(1)）
- 哈希表使用链地址法来解决碰撞问题
- rehash可以用于扩展或者收缩哈希表
- 对哈希表的rehash是分多次、渐进式进行的

&nbsp;

### **rehash**
**1. 触发条件：**

- 当新插入一个键值对的时候，根据used/size得到一个比例，如果这个比例超过阈值，就自动触发rehash过程
- 自然rehash:满足used/size >= 1 && dict_can_resize条件触发的
- 强制rehash:满足used/size >= dict_force_resize_ratio(默认为5)条件触发的。

**dict_can_resize标识当前是否能进行自然rehash的全局变量**

**2. 为啥需要两种类型的rehash？**

- 一切都是为了性能，不过这点考虑的是redis服务的整体性能
- 当redis使用后台子进程做一些工作，比如rdb save、aof rewrite等，为了最大化利用系统的copy on write机制，子进程会暂时将自然rehash关闭，这就是dict_can_resize的作用，如果允许对hash table进行resize操作的话，会造成大量的内存拷贝
- 当持久化任务完成后，将dict_can_resize设为true，就可以继续进行自然rehash
- 但是考虑另外一种情况，当现有字典的碰撞率太高了，那么就必须马上进行rehash防止再插入的值继续碰撞，这将浪费很长时间。所以超过dict_force_resize_ratio后，比如存的entry是slot数的五倍，则也会强制进行resize

**3. 渐进式rehash**

- 会在 rehash 的同时，保留新旧两个 hash 结构，查询时会同时查询两个 hash 结构，然后在后续的定时任务中以及 hash 操作指令中，循序渐进地将旧 hash 的内容一点点迁移到新的 hash 结构中
- 当搬迁完成了，就会使用新的hash结构取而代之；当 hash 移除了最后一个元素之后，该数据结构自动被删除，内存被回收
  - dictRehashStep：按照step进行的。当字典处于rehash状态(dict的rehashidx不为-1)，用户进行增删查改的时候会触发_dictRehashStep，这个函数就是将第一个索引不为空的全部节点迁移到ht[1]，因为一般情况下节点数目不会超过5(超过基本会触发强制rehash），所以成本很低，不会影响到响应时间。
  - dictRehashMilliseconds：这个相当于时间片轮转rehash，定时进行redis cron job来进行rehash。会在指定时间内对dict的rehashidx不为-1的字典进行rehash
  - 比如查找会先查ht[0]，没有再查ht[1]；字典的添加操作一律保存到ht[1]，保证只增不减

&nbsp;

## **1.4 集合对象**

**底层实现：**

- REDIS_ENCODING_INTSET（满足所有元素都是整数值&元素不超过512个）
- REDIS_ENCODING_HT(字典)

```
typedef struct intset {
    // 编码方式
    uint32_t encoding；
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组（int8_t 类型的，实现上它只是一个象征意义上的类型，到实际分配时候，会根据具体元素的类型选择合适的类型）
    int8_t contents[];
} intset;
```
**特点：**

- 保存有序、无重复的整数元素
- 根据元素的值自动选择对应的类型，但是intset只升级、不降级
- 升级会引起整个intset中的contents数组重新内存分配，并移动所有的元素（因为内存不一样了），所以复杂度为O(N)
- 因为int set是有序的，所以查找使用的是binary search

**集合的编码是决定于第一个添加进集合的元素，但也存在切换的阈值**

&nbsp;

## **1.5 有序集合对象**
**底层实现：**

- REDIS_ENCODING_ZIPLIST（元素数量小于128&所有元素成员的长度都小于64字节）
- REDIS_ENCODING_SKIPLIST

```
typedef struct zskiplistNode {
    // 成员对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;

/*
 * 跳跃表
 */
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```
**跳跃表是一种随机化数据结构（它的层是随机产生的），查找、添加、删除操作平均O(logN)级别的，最坏O(N)复杂度**

&nbsp;

## **1.6 实现场景**

- 消息队列 ：list(列表) 数据结构常用来作为异步消息队列使用，用blpop/brpop替代前面的lpop/rpop，阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。
- 延时队列：可以通过 Redis 的 zset(有序列表) 来实现。我们将消息序列化成一个字符串作为 zset 的value，这个消息的到期处理时间作为score，然后用多个线程轮询 zset 获取到期的任务进行处理

&nbsp;

# **二、内部机制实现**
## **2.1 过期键删除**
**Redis服务器实际使用的是惰性删除和定期删除两种策略**

- 惰性删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期，则删除；反之则取出该键
- 定期删除：每个一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于删除多少过期键，以及要检查多少个数据库，则有算法决定
  - Redis 默认会每隔100ms过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略
  - 从过期字典中随机 20 个 key
  - 删除这 20 个 key 中已经过期的 key
  - 如果过期的 key 比率超过 1/4，那就重复步骤 1
  - 同时，为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了扫描时间的上限，默认不会超过 25ms

**注意事项**

- 执行SAVE命令或BGSAVE命令所产生的新RDB文件不会包含已经过期的键
- 执行BGREWRITEAOF命令所产生的重写AOF文件不会包含已经过期的键
- 当一个过期键被删除之后，服务器会追加一条DEL命令到现有的AOF文件的末尾，显式地删除过期键
- 当主服务删除一个过期键之后，它会向所有从服务器发送一条DEL命令，显式地删除过期键
- 从服务器即使发现过期键也不会自作主张地删除它，而是等待主节点发来DEL命令，这样统一、中心化的过期键删除策略可以保证主从服务器数据的一致性

&nbsp;

## **2.2 Redis 内存淘汰机制**
**如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期 key 堆积在内存里，导致 redis 内存块耗尽了，怎么办？**
**maxmemory（最大使用内存）： 超出即需内存淘汰**

**maxmemory-policy：内存淘汰策略**

- noeviction：不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行（默认的淘汰策略）
- volatile-lru：尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失
- volatile-ttl：跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 的值，ttl 越小越优先被淘汰
- volatile-random：跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key
- allkeys-lru：区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰
- allkeys-random：跟上面一样，不过淘汰的策略是随机的 key

**LRU 算法 ： 双向链表 + 字典 实现**

近似 LRU 算法（redis采用）：给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。它的处理方式只有懒惰处理。当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次 LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。maxmemory_samples ：每次采样的个数，默认是5.

&nbsp;

## **2.3 持久化策略**
### **2.3.1 RDB**
#### **1. 实现**
**将数据库的快照以压缩的二进制文件的方式保存到磁盘**

- 一次全量备份，快照是内存数据的二进制序列化形式，在存储上非常紧凑，调用glibc的函数fork产生一个子进程，采用COW(Copy On Write) 机制来实现快照持久化，可以让 redis 保持高性能。
- 开启命令：save m n （表示m秒内数据集存在n次修改时）
- 手动触发：save阻塞当前Redis服务器；bgsave异步进行快照操作
- RDB文件的载入工作实在服务器启动时自动执行的，所以Redie并没有专门用于载入RDB文件的命令，只要Redis服务器在启动时检测到RDB文件存在，就会自动载入RDB文件，但是如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态

&nbsp;

### **2.3.2 AOF**
#### **1. 原理**
- 与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的
- RDB方式持久化的颗粒比较大，当服务器宕机时，到上次SAVE或BGSAVE后的所有数据都会丢失；而AOF的持久化颗粒比较细，当服务器宕机后，只有宕机之前没来得AOF的操作数据会丢失

&nbsp;

#### **2. 实现**
- 命令追加：服务器在执行完一个写命令后，会以协议的格式把其追加到服务器状态的aof_buf缓冲区末尾；
- 文件写入：redis服务器进程就是一个事件循环（loop），在每次事件循环结束，会根据配置文件中的appednfsync属性值决定是否将aof_buf中的数据写入到AOF文件中；
- 文件同步：将内存缓冲区的数据写到磁盘；（由于OS的特性导致）

&nbsp;

#### **3. appendfsync选项：**
- always：每执行一个命令保存一次。但是因为SAVE是Redis主进程执行的，所以在SAVE时候主进程阻塞，不再接受客户端的请求
- everysec：将aof_buf中所有内容写到AOF文件，如果上次同步AOF文件时间距当前时间超过1s，那么对AOF文件同步。因为SAVE是后台子线程调用的，所有主线程不会阻塞。
- no：不保存。这时候，调用flushAppendOnlyFile函数的时候WRITE都会执行（写入AOF程序的缓存），但SAVE会(写入磁盘)跳过，同步时间则有操作系统控制

&nbsp;

#### **4. 持久化效率和安全性（appendfsync控制）：**
- always每次都要写入、同步，所以其安全性最高，但SAVE是主进程进行，效率是最慢的
- everysec效率也足够快，并且在通常情况下， 这种模式最多丢失1 秒的命令数据， 并且SAVE由子进程完成，不会阻塞主进程
- no效率最高，但安全性比较差

&nbsp;

#### **5. 重写**
**BGREWRITE: 在后台（子进程）重写AOF, 不会阻塞工作线程，能正常服务，此方法最常用**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191225144557.png)

**原理**

- AOF文件重写并不需要对现有AOF文件进行任何读取、分析或者写入操作，这个功能是通过读取服务器当前的数据库状态来实现的。首先，从数据库中读取键现在的值，然后使用一条命令去记录键值对，代替之前记录这个键值对的多条命令，这就是AOF重写的原理
- Redis增加了一个AOF重写缓存区（在内存中），这个缓存区在创建子进程之后开始启用，当Redis主进程执行完一个写命令之后，它会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区。这样，当子进程完成AOF重写之后，它会给主进程发送一个信号，主进程接收信号后，会将AOF重写缓存中的内容全部写入新AOF文件中，然后对新AOF改名，原子地覆盖现有的AOF文件，完成新旧两个AOF文件的替换
- 官方优化：AOF重写期间，主进程跟子进程通过管道通信，主进程实时将新写入的数据发送给子进程，子进程从管道读出数据交缓存在buffer中，子进程等待存量数据全部写入AOF文件后，将缓存数据追加到AOF文件中，此方案只是解决阻塞工作线程问题，但占用内存过多问题并没有解决

&nbsp;

# **三、线程模型**
## **原理**
**redis 内部使用文件事件处理器 file event handler，这个文件事件处理器是单线程的，所以 redis 才叫做单线程的模型。它采用 IO 多路复用机制同时监听多个 socket，将产生事件的 socket 压入内存队列中，事件分派器根据 socket 上的事件类型来选择对应的事件处理器进行处理**

**文件事件处理器的结构包含 4 个部分：**

- 多个 socket
- IO 多路复用程序
- 文件事件分派器
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将产生事件的 socket 放入队列中排队，事件分派器每次从队列中取出一个 socket，根据 socket 的事件类型交给对应的事件处理器进行处理。

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191230141311.png)

## **redis单线程模型效率高原因**

- 纯内存操作，数据存放在内存中，内存的响应时间大约是100纳秒，这是Redis每秒万亿级别访问的重要基础
- 核心是非阻塞I/O，Redis采用epoll做为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接，读写，关闭都转换为了事件，不在I/O上浪费过多的时间
- C 语言实现，一般来说，C 语言实现的程序“距离”操作系统更近，执行速度相对会更快
- 单线程反而避免了多线程的频繁上下文切换问题，预防了多线程可能产生的竞争问题

## **参考**

- https://github.com/doocs/advanced-java/blob/master/docs/high-concurrency/redis-single-thread-model.md

&nbsp;

# **四、主从复制**

- Redis 主从架构，主从复制（异步）
- 主负责写，从节点负责读
- 主从架构 -> 读写分离 -> 水平扩容支撑读高并发

## **4.1 核心原理**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191225200931.png)

1. 当启动一个 slave node 的时候，它会发送一个 PSYNC 命令给 master node，如果这是 slave node 初次连接到 master node，那么会触发一次 full resynchronization 全量复制
2. 此时 master 会启动一个后台线程，开始生成一份 RDB 快照文件，同时还会将从客户端 client 新收到的所有写命令缓存在内存中
3. RDB 文件生成完毕后， master 会将这个 RDB 发送给 slave，slave 会先写入本地磁盘，然后再从本地磁盘加载到内存中
4. 接着 master 会将内存中缓存的写命令发送到 slave，slave 也会同步这些数据
5. slave node 如果跟 master node 有网络故障，断开了连接，会自动重连，连接之后 master node 仅会复制给 slave 部分缺少的数据

&nbsp;

## **4.2 主从复制的断点续传**

1. 从 redis2.8 开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份
2. master node 会在内存中维护一个 backlog，master 和 slave 都会保存一个 replica offset 还有一个 master run id，offset 就是保存在 backlog 中的
3. 如果 master 和 slave 网络连接断掉了，slave 会让 master 从上次 replica offset 开始继续复制，如果没有找到对应的 offset，那么就会执行一次 resynchronization（重新同步）

&nbsp;

## **4.3 完整复制流程**

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191225201610.png)

1. slave node 启动时，会在自己本地保存 master node 的信息，包括 master node 的host和ip，但是复制流程没开始
2. slave node 内部有个定时任务，每秒检查是否有新的 master node 要连接和复制，如果发现，就跟 master node 建立 socket 网络连接
3. 然后 slave node 发送 ping 命令给 master node，如果 master 设置了 requirepass，那么 slave node 必须发送 masterauth 的口令过去进行认证
4. master node 第一次执行全量复制，将所有数据发给 slave node
5. 而在后续，master node 持续将写命令，异步复制给 slave node

&nbsp;

## **4.4 全量复制**

1. master 执行 bgsave ，在本地生成一份 rdb 快照文件。
2. master node 将 rdb 快照文件发送给 slave node，如果 rdb 复制时间超过 60秒（repl-timeout），那么 slave node 就会认为复制失败，可以适当调大这个参数(对于千兆网卡的机器，一般每秒传输 100MB，6G 文件，很可能超过 60s)
3. master node 在生成 rdb 时，会将所有新的写命令缓存在内存中，在 slave node 保存了 rdb 之后，再将新的写命令复制给 slave node。
4. 如果在复制期间，内存缓冲区持续消耗超过 64MB，或者一次性超过 256MB，那么停止复制，复制失败。
```
client-output-buffer-limit slave 256MB 64MB 60
```
5. slave node 接收到 rdb 之后，清空自己的旧数据，然后重新加载 rdb 到自己的内存中，同时基于旧的数据版本对外提供服务。
6. 如果 slave node 开启了 AOF，那么会立即执行 BGREWRITEAOF，重写 AOF。

&nbsp;

## **4.5 增量复制**

1. 如果全量复制过程中，master-slave 网络连接断掉，那么 slave 重新连接 master 时，会触发增量复制。
2. master 直接从自己的 backlog 中获取部分丢失的数据，发送给 slave node，默认 backlog 就是 1MB。
3. master 就是根据 slave 发送的 psync 中的 offset 来从 backlog 中获取数据的。

&nbsp;

## **4.6 heartbeat**
**主从节点互相都会发送 heartbeat 信息**

- master 默认每隔 10秒 发送一次 heartbeat
- slave node 每隔 1秒 发送一个 heartbeat。

**异步复制**

master 每次接收到写命令之后，先在内部写入数据，然后异步发送给 slave node。

&nbsp;

# **五、高可用**
## **5.1 哨兵的介绍**
**sentinel，中文名是哨兵。哨兵是 redis 集群机构中非常重要的一个组件，主要有以下功能：**

- 集群监控：负责监控 redis master 和 slave 进程是否正常工作
- 消息通知：如果某个 redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址

**哨兵用于实现 redis 集群的高可用，本身也是分布式的，作为一个哨兵集群去运行，互相协同工作**

- 故障转移时，判断一个 master node 是否宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题
- 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑爹了

&nbsp;

## **5.2 哨兵的核心知识**

- 哨兵至少需要 3 个实例，来保证自己的健壮性
- 哨兵 + redis 主从的部署架构，**是不保证数据零丢失的**，只能保证 redis 集群的高可用性。
- 对于哨兵 + redis 主从这种复杂的部署架构，尽量在测试环境和生产环境，都进行充足的测试和演练。

&nbsp;

## **5.3 redis 哨兵主备切换的数据丢失问题**

### **1. 两种丢失情况**

- 异步复制导致的数据丢失
  - master->slave 的复制是异步的，所以可能有部分数据还没复制到 slave，master 就宕机了，此时这部分数据就丢失了
- 脑裂导致的数据丢失
  - 某个 master 所在机器突然脱离了正常的网络，跟其他 slave 机器不能连接，但是实际上 master 还运行着。此时哨兵可能就会认为 master 宕机了，然后开启选举，将其他 slave 切换成了 master。这个时候，集群里就会有两个 master ，也就是所谓的脑裂
  - 此时虽然某个 slave 被切换成了 master，但是可能 client 还没来得及切换到新的 master，还继续向旧 master 写数据。因此旧 master 再次恢复的时候，会被作为一个 slave 挂到新的 master 上去，自己的数据会清空，重新从新的 master 复制数据。而新的 master 并没有后来 client 写入的数据，因此，这部分数据也就丢失了

&nbsp;

### **2. 数据丢失问题的解决方案**
```
min-slaves-to-write 1
min-slaves-max-lag 10
```
**表示，要求至少有 1 个 slave，数据复制和同步的延迟不能超过 10 秒；如果说一旦所有的 slave，数据复制和同步的延迟都超过了 10 秒钟，那么这个时候，master 就不会再接收任何请求了**

- 减少异步复制数据的丢失
  - 有了 min-slaves-max-lag 这个配置，就可以确保说，一旦 slave 复制数据和 ack 延时太长，就认为可能 master 宕机后损失的数据太多了，那么就拒绝写请求，这样可以把 master 宕机时由于部分数据未同步到 slave 导致的数据丢失降低的可控范围内。
- 减少脑裂的数据丢失
  - 如果一个 master 出现了脑裂，跟其他 slave 丢了连接，那么上面两个配置可以确保说，如果不能继续给指定数量的 slave 发送数据，而且 slave 超过 10 秒没有给自己 ack 消息，那么就直接拒绝客户端的写请求。因此在脑裂场景下，最多就丢失 10 秒的数据。

&nbsp;

## **5.4 sdown 和 odown 转换机制**

- sdown 是主观宕机，就一个哨兵如果自己觉得一个 master 宕机了，那么就是主观宕机
  - sdown 达成的条件很简单，如果一个哨兵 ping 一个 master，超过了 is-master-down-after-milliseconds 指定的毫秒数之后，就主观认为 master 宕机了
- odown 是客观宕机，如果 quorum 数量的哨兵都觉得一个 master 宕机了，那么就是客观宕机
  - 如果一个哨兵在指定时间内，收到了 quorum 数量的其它哨兵也认为那个 master 是 sdown 的，那么就认为是 odown 了

&nbsp;

## **5.5 哨兵集群的自动发现机制**

- 哨兵互相之间的发现，是通过 **redis 的 pub/sub（发布订阅） 系统实现的**，每个哨兵都会往__sentinel__:hello 这个 channel 里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在
- 每隔两秒钟，每个哨兵都会往自己监控的某个 master+slaves 对应的__sentinel__:hello channel 里发送一个消息，内容是自己的 host、ip 和 runid 还有对这个 master 的监控配置
- 每个哨兵也会去监听自己监控的每个 master+slaves 对应的__sentinel__:hello channel，然后去感知到同样在监听这个 master+slaves 的其他哨兵的存在
- 每个哨兵还会跟其他哨兵交换对 master 的监控配置，互相进行监控配置的同步

&nbsp;

## **5.6 slave->master 选举算法**
**基于Raft算法**

**考虑因素**

- 跟 master 断开连接的时长
- slave 优先级
- 复制 offset
- run id

**排序算法**

- 先看是否超过(down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state，那么 slave 就被认为不适合选举为 master
- 按照 slave 优先级进行排序，slave priority 越低，优先级就越高
- 如果 slave priority 相同，那么看 replica offset，哪个 slave 复制了越多的数据，offset 越靠后，优先级就越高
- 如果上面两个条件都相同，那么选择一个 run id 比较小的那个 slave

&nbsp;

## **5.7 configuration epoch**

- 哨兵会对一套 redis master+slaves 进行监控，有相应的监控的配置
- 执行切换的那个哨兵，会从要切换到的新 master（salve->master）那里得到一个 configuration epoch，这就是一个 version 号，每次切换的 version 号都必须是唯一的
- 如果第一个选举出的哨兵切换失败了，那么其他哨兵，会等待 failover-timeout 时间，然后接替继续执行切换，此时会重新获取一个新的 configuration epoch，作为新的version号

&nbsp;

## **5.8 configuration 传播**
- 哨兵完成切换之后，会在自己本地更新生成最新的 master 配置，然后同步给其他的哨兵，就是通过之前说的 pub/sub 消息机制。
- 这里之前的 version 号就很重要了，因为各种消息都是通过一个 channel 去发布和监听的，所以一个哨兵完成一次新的切换之后，新的 master 配置是跟着新的 version 号的，其他的哨兵都是根据版本号的大小来更新自己的 master 配置的

&nbsp;

# **六、集群模式**
**Redis Cluster（稳定版本5.0.4，3.0之后支持集群）**
**遇到 单机内存、并发、流量 等瓶颈时，可以采用 Cluster 架构方案达到 负载均衡 的目的**

## **6.1 原理**

- Redis Cluster 是Redis的集群实现，内置数据自动分片机制，每个 master 上放一部分数据
- 集群内部将所有的key映射到16384个Slot中，集群中的每个Redis Instance负责其中的一部分的Slot的读写
- 集群客户端连接集群中任一Redis Instance即可发送命令，客户端连接集群时会得到一份集群slot信息，当客户端要查找某个 key 时，可以直接定位到目标节点
- 当Redis Instance收到自己不负责的Slot的请求时，会将负责请求Key所在Slot的Redis Instance地址返回给客户端，客户端收到后自动将原请求重新发往这个地址，对外部透明
- 一个Key到底属于哪个Slot由crc16(key) % 16384 决定
- 在 redis cluster 架构下，每个redis要放开两个端口号，比如一个是6379，另外一个就是加1w的端口号，比如16379，用来进行节点间通信的

&nbsp;

## **6.2 节点间的内部通信机制**

- redis cluster 节点间采用 gossip 协议进行通信
- gossip协议：所有节点都持有一份元数据，不同的节点如果出现了元数据的变更，就不断将元数据发送给其它的节点，让其它节点也进行元数据的变更
- gossip 好处在于元数据的更新比较分散，不是集中在一个地方，更新请求会陆陆续续打到所有节点上去更新，降低了压力；不好在于，元数据的更新有延时，可能导致集群中的一些操作会有一些滞后

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191228143600.png)

&nbsp;

## **6.3 gossip 协议**
**gossip 协议包含多种消息，包含 ping,pong,meet,fail 等等**

- ping：每个节点都会频繁给其它节点发送 ping，其中包含自己的状态还有自己维护的集群元数据，互相通过 ping 交换元数据
- pong：返回 ping 和 meeet，包含自己的状态和其它信息，也用于信息广播和更新
- meet：某个节点发送 meet 给新加入的节点，让新节点加入集群中，然后新节点就会开始与其它节点进行通信
- fail：某个节点判断另一个节点 fail 之后，就发送 fail 给其它节点，通知其它节点说，某个节点宕机啦

&nbsp;

## **6.4 槽位迁移**

- Redis迁移的单位是槽，Redis 一个槽一个槽进行迁移
- 槽在原节点的状态为migrating，在目标节点的状态为importing
- 从源节点获取内容 => 存到目标节点 => 从源节点删除内容
- 迁移过程是同步的，节点的主线程会处于阻塞状态，直到key被成功删除

&nbsp;

## **6.5 redis cluster 的高可用与主备切换原理**
### **1. 判断节点宕机**

- 如果一个节点认为另外一个节点宕机，那么就是 pfail，主观宕机
- 如果多个节点都认为另外一个节点宕机了，那么就是 fail，客观宕机，跟哨兵的原理几乎一样，sdown，odown
- 在 cluster-node-timeout 内，某个节点一直没有返回 pong，那么就被认为 pfail
- 如果一个节点认为某个节点 pfail 了，那么会在 gossip ping 消息中，ping 给其他节点，如果超过半数的节点都认为 pfail 了，那么就会变成 fail

&nbsp;

### **2. 从节点过滤**

- 对宕机的 master node，从其所有的 slave node 中，选择一个切换成 master node
- 检查每个 slave node 与 master node 断开连接的时间，如果超过了 cluster-node-timeout * cluster-slave-validity-factor，那么就没有资格切换成 master

&nbsp;

### **3. 从节点选举**

- 每个从节点，都根据自己对 master 复制数据的 offset，来设置一个选举时间，offset 越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举。
- 所有的 master node 开始 slave 选举投票，给要进行选举的 slave 进行投票，如果大部分 master node（N/2 + 1）都投票给了某个从节点，那么选举通过，那个从节点可以切换成 master
- 从节点执行主备切换，从节点切换为主节点。

&nbsp;

# **七、缓存问题**

## **7.1 缓存雪崩**

- **定义：某一时刻发生大规模的缓存失效的情况，例如缓存服务宕机**
- **问题：大量的请求进来直接打到DB上面，DB直接挂掉**

**解决方案**

- 事前：redis 高可用，主从 + 哨兵，redis cluster，避免全盘崩溃
- 事中：本地缓存 + hystrix 限流&降级，避免 MySQL 被打死
- 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据

&nbsp;

## **7.2 缓存击穿**

- **定义：大量的请求同时查询一个 key 时，此时这个key正好失效了，就会导致大量的请求都打到数据库上面去**
- **问题：造成某一时刻数据库请求量过大，压力剧增**

**解决方案**

- 若缓存的数据是基本不会发生更新的，则可尝试将该热点数据设置为永不过期
- 若缓存的数据更新不频繁，且缓存刷新的整个流程耗时较少的情况下，则可以采用基于 redis、zookeeper 等分布式中间件的分布式互斥锁，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存
- 若缓存的数据更新频繁或者缓存刷新的流程耗时较长的情况下，可以利用定时线程在缓存过期前主动的重新构建缓存或者延后缓存的过期时间，以保证所有的请求能一直访问到对应的缓存

&nbsp;

## **7.3 缓存穿透**

- **定义：业务系统访问缓存和数据库都查询不到的数据，例如黑客攻击，逻辑代码错误等**
- **问题：请求都会落到数据库中，数据库压力剧增，可能导致数据库挂掉**

**解决方案**

- 缓存空值
  - 将数据库查询结果为空的key也存储在缓存中
  - 场景：空数据的key数量有限、key重复请求概率较高
- BloomFilter
  - BloomFilter存储目前数据库中存在的所有key
  - 当业务系统有查询请求的时候，首先去BloomFilter中查询该key是否存在。若不存在，直接返回null。若存在，则继续执行后续的流程。
  - 场景：空数据的key各不相同、key重复请求概率低

&nbsp;

## **7.4 热点key**

- **定义：缓存会设定一个失效时间，但一些请求量极高的热点数据而言，一旦过了有效时间，此刻将会有大量请求落在数据库上**
- **问题：请求都会落到数据库中，数据库压力剧增，可能导致数据库挂掉**

**解决方案**

- 互斥锁
  - 在第一个请求去查询数据库的时候对他加一个互斥锁，其余的查询请求都会被阻塞住，直到锁被释放，从而保护数据库。
  - 但是也是由于它会阻塞其他的线程，此时系统吞吐量会下降。需要结合实际的业务去考虑是否要这么做。
- 设置不同的失效时间
  - 在设置缓存过期时间的时候，让失效的时间错开。如：在一个基础时间上加/减一个随机数

&nbsp;

## **7.5 大key**

- **定义：key所对应的存储的value过大**
- **问题：**
  - 如果是String类型的大key，由于redis是单线程运行，所以读写大key会导致redis性能下降，甚至阻塞服务
  - 如果是hash、set、list、zset等类型的key，删除key会严重阻塞redis进程，因为删除时间复杂度是O(n)
  - 大key也会造成集群数据倾斜

**解决方案**

- 单个简单key的value过大
  - 拆分成多个key，用批量命令mget获取，这样数据会分散到不同节点上，流量均摊
  - 也可以把这个key拆分到hash中，用hget获取
- hash、set等集合类型的存储元素过多
  - 那就把单个集合，拆成多个集合，然后根据具体的key路由到相应的集合上

**最佳实践**

- String类型大小控制在10KB以内
- 集合元素不要超过5000个

&nbsp;

## **7.6 本地缓存**
**guava cache**

https://github.com/doocs/advanced-java/tree/master/docs/high-concurrency
