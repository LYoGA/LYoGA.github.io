---
title: "Jvm总结"
date: 2019-06-07T16:12:03+08:00
lastmod: 2019-06-07T16:12:03+08:00
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

# **一、JVM内存结构**#

## **1.1 JVM整体由4部分组成**##

1. 类加载子系统
2. 执行引擎
3. 运行时数据区
4. 内存回收

&nbsp;

## **1.2 运行时数据区：**##
1. 程序计数器：线程所执行的字节码的行号指示器
2. JAVA虚拟机栈：每个方法被执行的时候都会同时创建一个栈帧用于存放局部变量表（编译时确定大小）、操作栈、动态链接、方法出口等信息
3. 本地方法栈：针对native方法
4. 堆：几乎所有的对象实例都在堆分配（栈上分配&TLAB）
  - 栈上分配：针对那些作用域不会逃逸出方法的对象，在分配内存时不在将对象分配在堆内存中，而是将对象属性打散后分配在栈上，这样随着方法的调用结束，栈空间的回收就会随着将栈上分配的打散后的对象回收掉，不再给gc增加额外的无用负担，从而提升应用程序整体的性能（开启逃逸分析&标量替换）
  - TLAB(Thread Local Allocation Buffer)：线程本地分配缓存，是在为新对象分配内存空间时，让每个Java应用线程能在使用自己专属的分配指针来分配空间，均摊对GC堆（eden区）里共享的分配指针做更新而带来的同步开销
5. 方法区：用于存储已被虚拟机加载的类信息（Class）、常量、静态变量、即时编译器编译后的代码等数据
  - Class文件常量池：字面量&符号引用
  - 运行时常量池：存储Java class文件常量池中的符号信息；运行时常量池中保存着一些class文件中描述的符号引用，同时在类加载的“解析阶段”还会将这些符号引用所翻译出来的直接引用(直接指向实例对象的指针)存储在运行时常量池中
  - 字符串常量池：字符串常量池是JVM所维护的一个字符串实例的引用表，在HotSpot VM中，它是一个叫做StringTable的全局表。在字符串常量池中维护的是字符串实例的引用，底层C++实现就是一个Hashtable。
6. 元空间：元空间并不在虚拟机中，而是使用本地内存，本地内存消耗完也会报OOM，类及相关的元数据的生命周期与类加载器的一致
  - 本地内存分配
  - 使用块分配器，线性分配，以类加载器为单位，块的大小取决于类加载器的类型

&nbsp;

## **1.3 JDK7、8区别**##
- JDK7中，存储在永久代的部分数据就已经转移到了Java Heap或者是 Native Heap，比如符号引用(Symbols)转移到了native heap，字符串常量池(interned strings)转移到了java heap
- JDK8中，永久代已经完全被元空间取代
  - PermGen内存经常溢出（OOM）&移除PermGen可以促进HotSpot JVM与JRockit VM的融合，因为JRockit没有永久代

&nbsp;

# **二、内存回收** #
## **2.1 对象存活判定**##
- 引用计数：循环引用，造成内存泄漏
- 可达性算法分析：GC Roots作为起点，向下搜索，判断哪些对象是没有任何引用链，是不可达的
  - GC Roots对象：
      - JAVA栈（栈帧中的局部变量表，Local Variable Table）中引用的对象
      - 本地方法栈中JNI（即一般说的Native方法）引用的对象
      - 方法区中类静态变量引用的对象
      - 方法区中常量引用的对象
  - 选取依据：
      - 确定是存活的对象集
      - 全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的局部变量表）
      - JAVA栈、方法区和本地方法栈不被GC所管理

&nbsp;

## **2.2 两次标记与finalize()**##
- 真正宣告一个对象死亡，有可能经历两次标记过程
- 对象在经历可达性分析之后，无任何引用链，那它将会被第一次标记并且进行一次筛选，筛选条件是此对象是否有必要执行finalize方法
- 对象无覆盖finalize方法或者已经调用过了，就直接回收
- 如果有必要执行，将对象放到一个叫做F-Queue的队列中，之后由虚拟机自动建立的低优先级的Finalize线程执行，不会承诺会等待运行结束，不可控的
- finalize方法是对象最后一次逃脱机会，稍后GC将对F-Queue队列进行小规模二次标记，看是否有引用链
- 不建议使用，建议忘掉这个方法

&nbsp;

## **2.3 回收方法区**##
- 该类所有实例已被回收
- 加载该类的类加载器也被回收
- 类对应的class对象，无任何地方被引用，主要无法通过反射获得
- 元空间：当一个类加载器被垃圾回收器标记为不再存活，其对应的元空间会被回收

&nbsp;

## **2.4 垃圾回收算法**##
- 标记-清除：简单直接，但会造成内存碎片
- 复制：内存对半分，简单，不会产生内存碎片，但会浪费部分内存空间
- 标记-整理：压缩空间碎片，不会产生内存碎片
- 分代收集
  - 新生代：Minor GC，复制算法，eden：from survivor：to survivor = 8：1：1，浪费10%新生代容量，比例可调
      - Minor GC将eden与from survivor中还存活的对象复制到to survivor，然后清理eden与from survivor
      - 交换from survivor与to survivor标签

&nbsp;

## **2.5 HotSpot算法实现**##
1. 枚举GC Roots
2. GC停顿：可达性分析工作必须在一个能确保一致性的快照中进行（STW - Stop The World）
3. 准确式GC与OopMap
  - 准确式内存管理，虚拟机可以知道内存中某个位置的数据具体是什么类型
  - OopMap数据结构存储这些信息，类加载完成，HotSpot就把对象什么偏移量上是什么类型算出来，JIT编译过程中，也会在特定位置记录下栈和寄存器中哪些位置是引用
4. 安全点：进行GC时程序停顿的位置
5. 安全区域：安全区域是指在一段代码片段之中，引用关系不会发生变化，在这个区域中的任意地方开始GC都是安全的

&nbsp;

## **2.6 内存分配策略**##
**对象的内存分配：栈上分配 -> TLAB分配 -> 堆上分配**

1. 对象优先在Eden区分配
2. 大对象直接进入老年代
3. 长期存活对象将进入老年代，默认15岁，经过15次移动
4. 动态年龄对象判定：如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄
5. 空间分配担保：进行Minor GC之前，会检查老年代可用连续空间是否大于新生代所有对象总和，如果不成立就看HandlePromotionFailure设置值是否允许担保失败
  - 如果允许，检查当前老年代可用连续空间是否大于历次晋升老年代对象的平均值，如果大于，将尝试进行一次Minor GC，如果小于，则Full GC

&nbsp;

## **2.7 Full GC触发条件**##
1. 调用System.gc()
2. 老年代空间不足
3. 空间分配担保失败
4. 1.7以及以前永久代空间不足
5. CMS GC过程中同时有对象放入老年代，而老年代空间不足（有可能是浮动垃圾过多导致），便会报Concurrent Mode Failure错误，并触发Full GC

&nbsp;

# **三、垃圾回收器**#

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191024114122.png)

## **3.1 新生代垃圾收集器**##
1. Serial收集器：复制算法、单线程、独占式、串行
2. ParNew：复制算法、多线程、独占式，与CMS配合工作、并行
3. Parallel Scavenge：复制算法、多线程、独占式、并行；精准控制吞吐量、GC时间；可配置GC时间的最大值、直接设置吞吐量大小

&nbsp;

## **3.2 老年代垃圾收集器**##
1. Serial Old：标记整理、单线程、独占式、串行；作为CMS的后备预案，Concurrent Mode Failure时使用
2. Parallel Old：标记整理、多线程、独占式、并行
3. CMS（Concurrent Mark-Sweep）：标记清除、多线程、非独占式（是以牺牲吞吐量为代价来获得最短回收停顿时间的垃圾回收器）
  - 优点：并发收集、低停顿
  - 缺点：
      - 对CPU资源非常敏感（CMS默认启动的回收线程数是（CPU数量+3）/4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降）
      - 无法处理浮动垃圾，并发清理时用户线程还在运行
      - 标记清除算法存在内存碎片

&nbsp;

## **3.3 CMS**##

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191024114441.png)

### **3.3.1 运行周期**###

- 初始标记：仅标记GC Roots能直接关联到的对象，需要STW
  - 这里的GC root不包括yong gen，而只是stack、register、globals这些常规的，因为在并发标记阶段，CMS会顺着初始根集合把young gen中的h活对象都遍历
  - 所以从初始标记+并发标记结合在一起的角度看，young gen仍然是根集合的一部分
- 并发标记：进行GC Roots Tracing过程，标记可达对象，耗时长
  - 并发预清理：并发查找在做并发标记阶段时从年轻代晋升到老年代的对象或老年代新分配的对象(大对象直接进入老年代)或被用户线程更新的对象，为了减少重新标记阶段的工作量，会把所在的Card标记为Dirty
  - 预清理阶段可能会出现Young GC
- 重新标记：修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，比初始标记时间长点，但远比并发标记时间短，需要STW
  - 遍历新生代对象，重新标记
  - 根据GC Roots，重新标记
  - 遍历老年代的Dirty Card（Card Table + Mod Union Table），重新标记
- 并发清除：清除无用对象
- 并发重置

&nbsp;

### **3.3.2 无法处理浮动垃圾的补偿策略**###

- jdk1.5默认使用68%空间，触发Full GC；jdk1.6，默认92%；-XX:CMSInitiatingOccupancyFraction调整百分比
- 后备方案：临时启动Serial Old收集器

&nbsp;

### **3.3.3 内存碎片补偿策略**###

- -XX:+UseCMSCompactAtFullCollection开关参数（默认开启）：用于CMS收集器顶不住要进行Full GC时开启内存碎片整合过程，内存整理过程是无法并发的
- -XX:CMSFullGCsBeforeCompaction：用于设置多少次不压缩的Full GC之后，跟着来一次带压缩的（默认0，即每次都进行碎片压缩）

&nbsp;

### **3.3.4 并发标记如何实现的**###

- 前提：并发标记在运行期间可能发生新生代的对象晋升到老年代、或者是直接在老年代分配对象、或者更新老年代对象的引用关系等等，对于这些对象，都是需要进行重新标记的，否则有些对象就会被遗漏，发生漏标的情况
- 解决方法：incremental update write barrier（增量式GC），也叫做insertion barrier
  - Incremental update的write barrier会拦截所有新插入的引用关系，并且按需要记录新的引用关系，其实只是在card table记录一下改变的引用的出发端对应的card
  - 有了Card Table还需要一个Mod Union Table的原因是，本身Card Table只有一份，既要来支持young gc又要来支持CMS的话，因为本身young gc过程中都涉及重置和重新扫描Card Table，这样是满足了young gc需求，但却破坏了CMS的需求--CMS需要的信息可能被young gc重置了，所以为了避免信息丢失，就在card table之后又加了一个bitmap叫做mod union table
  - CMS在并发标记过程中，每当发生一次young gc，当young gc要重置card table里的某个记录时，就会更新mod union table对应的bit
  - CMS remark的时候，当时的card table外加mod-union table就足以记录在并发标记过程中old gen发生的所有引用变化了
  - HotSpot VM只使用了old gen部分的card table，也就是说只关心old -> ?的引用。这是因为一般认为young gen的引用变化率（mutation rate）非常高，其对应的card table部分可能大部分都是dirty的，要把young gen当作root的时候与其扫描card table还不如直接扫描整个young gen

- 三色标记法
  - 黑色black，表明对象被collector访问过，属于可到达对象
  - 灰色gray，也表明对象被访问过，但是它的子节点还没有被scan到
  - 白色white，表明没有被访问到，如果在本轮遍历结束时还是白色，那么就会被收回

&nbsp;

### **3.3.5 重新标记阶段，如果重新扫描根集合，那岂不是initial mark和concurrent mark两个阶段所做的工作都白做了，整个对象关系图又需要重新标记一遍？**###

其实不是的，因为每个root的扫描过程在触及到一个marked对象的时候就会终止，而concurrent mark阶段正常情况应该已经完成了绝大部分可达对象的标记。

&nbsp;

### **3.3.6 出现老年代引用新生代的对象，GC怎么处理？**###

- **JVM采用了Card Marking(卡片标记)的方法，避免了在做Young GC时需要对整个老年代扫描**
- 将老年代按照一定大小分片(Cards Table)，每一片对应Cards中的一项，如果老年代的对象发生了修改，或者老年代对象指向了新生代对象，就把这个老年代对象所在的Card标记为脏dirty
- Young GC时，dirty card加入待扫描的GC Roots范围，避免扫描整个老年代

&nbsp;

### **3.3.7 与应用程序一起运行，为何采用清除算法？**###

CMS主要关注低延迟，因而采用并发方式，清理垃圾时，应用程序还在运行，如何采用压缩算法，则涉及到要移动应用程序的存活对象，此时不停顿，是很难处理的，一般需要停顿下，移动存活对象，再让应用程序继续运行，但这样停顿时间变长，延迟变大，所以CMS采用清除算法。

&nbsp;

### **3.3.8 CMS参数调优**###


&nbsp;

## **3.4 G1收集器**##
### **3.4.1 内存特点**

- 粒度：Region，通过参数-XX:G1HeapRegionSize，可以设置大小(1M、2M、4M、8M、16M、32M)
- 分代策略：仍然是分代回收，新生代 + 老年代
- Region分类：
  - Eden
  - Survivor
  - Old
  - Humongous：大对象Region，单个对象超过region的50%，如果超过一个Region，则会占用连续的Region
  - Free：空闲Region、可用Region
- Region的分类，动态可变：Region被释放后，是Free Region，后续可能被作为 Eden or Survivor or Old or Humongous 分区使用

&nbsp;

### **3.4.2 关联术语：**

- CSet：Collection Set，待回收的Region集合
- RSet：Remembered Set，引用当前Region的其他Region集合，是points-in的，每个Region都有一个RSet；在JDK10中，只有部分Region才存在RSet

&nbsp;

### **3.4.3 关键用途：**

- RSet：指向「当前 Region」 的「其他 Region 集合」，有两个典型应用场景
- young gc：指向「Eden」和「Survivor」Region 的 「Old」和「Humongous」Region集合，避免全堆扫描
- mixed gc：确定「Old」Region之间的相关引用，确定Region的回收价值

&nbsp;

### **3.4.4 G1执行过程**
**从全局来看看，G1垃圾回收可以分为两大部分（而这两部分可以相对独立的执行）：**

- 全局并发标记（Global Concurrent Marking）
- 拷贝存活对象（Evacuation）

&nbsp;

### **3.4.5 Global Concurrent Marking**

**Global Concurrent Marking 是基于 SATB 形式的并发标记，SATB（snapshot-at-the-beginning）是一种比CMS收集器更快的算法。Global Concurrent Marking 具体分为下面几个阶段：**

1. 始标记（STW initial marking）：扫描根集合，标记所有从根集合可直接到达的对象并将它们的字段压入扫描栈。在分代式G1模式中，初始标记阶段借用 Young GC 的暂停，因而没有额外的、单独的暂停阶段
2. 并发标记（concrrent marking）：这个阶段可以并发执行，GC 线程 不断从扫描栈取出引用，进行递归标记，直到扫描栈清空
3. 最终标记（STW final marking，在实现中也叫Remarking）：重新标记写入屏障（ Write Barrier）标记的对象，这个阶段也进行弱引用处理（reference processing）,G1的SATB设计在remark阶段则只需要扫描剩下的satb_mark_queue
4. 清理（STW cleanup）：统计每个 Region 被标记为活的对象有多少，如果发现完全没有活对象的 Region 就会将其整体回收到可分配 Region 列表中

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191029151511.png)

**1. SATB算法实现**
**SATB利用pre write barrier将所有即将被删除的引用关系的旧引用记录下来，最后以这些旧引用为根STW地重新扫描一遍即可避免漏标问题**

SATB抽象的说就是在一次GC开始的时候是活的对象就被认为是活的，此时的对象图形成一个逻辑“快照”（snapshot）；然后在GC过程中新分配的对象都当作是活的。其它不可到达的对象就是死的了。

```
G1CMBitMap * _prev_mark_bitmap; /* 全局的bitmap，存储preTAMS偏移位置，也即当前标记的对象的地址（初始值是对应上次已经标记完成的地址） */
G1CMBitMap * _next_mark_bitmap; /* 全局的bitmap，存储nextTAMS偏移位置。标记过程不断移动，标记完成后会和_prev_mark_bitmap互换。 */        
```
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20191030192023.png)
其中top是该region的当前分配指针，[bottom, top)是当前该region已用（used）的部分，[top, end)是尚未使用的可分配空间（unused）

1. [bottom, prevTAMS): 这部分里的对象存活信息可以通过prevBitmap来得知
2. [prevTAMS, nextTAMS): 这部分里的对象在第n-1轮concurrent marking是隐式存活的
3. [nextTAMS, top): 这部分里的对象在第n轮concurrent marking是隐式存活的

**2. pre wirte barrier实现**

- 以SATB write barrier为例，每个Java线程有一个独立的、定长的SATBMarkQueue，mutator在barrier里只把old_value压入该队列中。
- 一个队列满了之后，它就会被加到全局的SATB队列集合SATBMarkQueueSet里等待处理，然后给对应的Java线程换一个新的、干净的队列继续执行下去。
- 并发标记（concurrent marker）会定期检查全局SATB队列集合的大小。
- 当全局集合中队列数量超过一定阈值后，concurrent marker就会处理集合里的所有队列：把队列里记录的每个oop都标记上，并将其引用字段压到标记栈（marking stack）上等后面做进一步标记。

**3. 总结流程：**

- 在开始标记的时候生成一个快照图标记存活对象
- 在并发标记的时候所有被改变的对象入队（在write barrier里把所有旧的引用所指向的对象都变成非白的）
- 可能存在游离的垃圾，将在下次被收集

**4. RSet如何更新?**

**G1中采用post-write barrier和concurrent refinement threads实现了RSet的更新**

- 在赋值动作前后，JVM分别插入pre-write barrier和post-wirte barrier
- post-wirte barrier的最终动作如下
  - 找到该字段所在的位置（Card），并设置为dirty_card
  - 每个java线程都有一个dirty card queue，把该card插入队列中；除了每个线程自带的dirty card queue，还有一个全局共享的queue
- 赋值动作完成后，RSet的更新动作交给ConcurrentG1RefineThread并发完成
  - 每当全局queue超过一个阈值，ConcurrentG1RefineThread会取出若干个队列，遍历每个队列中记录的card，并行处理

&nbsp;

### **3.4.6 Evacuation**
**Evacuation阶段是全暂停的。它负责把一部分 Region 里的活对象拷贝到空 Region 里去，然后回收原本的 Region 的空间**

- G1虽然会mark整个堆，但并不evacuate所有有活对象的region
- 通过只选择收益高的少量region来evacuate，这种暂停的开销就可以（在一定范围内）可控
- 每次evacuate的暂停时间应该跟一般GC的young GC类似。所以G1把自己标榜为“软实时”（soft real-time）的GC

&nbsp;

### **3.4.7 G1分代回收**

**可以分为 Young GC 和 Mixed GC 两种类型：**

- Young GC：选定所有 新生代 里的 Region ，通过控制 新生代 的 Region 个数来控制 Young GC 的开销
- Mixed GC：选定所有 新生代 里的 Region ，外加根据 Global Concurrent Marking 统计得出收集收益高的若干老年代 Region ，在用户指定的开销目标范围内尽可能选择收益高的老年代 Region

1. 分代式G1的正常工作流程就是在 Young GC 与 Mixed GC之间视情况切换，背后定期做做全局并发标记。Initial marking 默认搭在 Young GC 上执行；当全局并发标记正在工作时，G1 不会选择做 Mixed GC，反之如果有 Mixed GC 正在进行中 G1 也不会启动 initial marking
2. 同 CMS 一样，当所有 Eden Region 被耗尽无法申请内存时，Young GC 就会被触发
3. 如果mixed GC实在无法跟上程序分配内存的速度，导致old gen填满无法继续进行mixed GC，**就会切换到G1之外的serial old GC来收集整个GC heap**（注意，包括young、old、perm），这才是真正的full GC


**一个假想的混合的STW时间线：**
```
-> young GC
-> young GC
-> young GC
-> young GC + initial marking
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
(... concurrent marking ...)
-> young GC (... concurrent marking ...)
-> final marking
-> cleanup
-> mixed GC
-> mixed GC
-> mixed GC
...
-> mixed GC
-> young GC + initial marking
(... concurrent marking ...)
...
```

**G1参数调优**

> -XX:MaxGCPauseMillis=n

设置垃圾收集暂停时间最大值指标，注意这个目标不一定能满足，Java虚拟机将尽最大努力实现它，不建议设置得过小（ < 50ms ）;

> -XX:InitiatingHeapOccupancyPercent=n

触发并发垃圾收集周期的整个堆空间的占用比例。

**最佳实践1：不要设置新生代大小**
通过 -Xmn 显式地设置新生代大小会干扰 G1 的垃圾回收策略：

- 设置的最大暂停时间指标将不再有效，事实上，设置新生代大小后，将不会启用暂停时间目标
- G1收集器将不能按需调整新生代的大小空间

**最佳实践2：避免晋升失败带来的副作用**

晋升失败后，如果此时 JVM 堆内存也耗尽了，就会出现 Evacuation Failure，在GC日志里将会看到to-space overflow的日志。Evacuation Failure的开销是巨大的，为了避免这种情况，可以执行下面的步骤：

- 增加 -XX:G1ReservePercent 选项的值（并相应增加总的堆大小），为“目标空间”增加预留内存量
- 通过减少 -XX:InitiatingHeapOccupancyPercent 提前启动标记周期
- 通过设置-XX:ConcGCThreads=n增加并行标记线程的数量

&nbsp;

### **3.4.8 易混淆的概念**
**1. CMS并发标记和G1并发标记的区别**

1. CMS采用的incremental update write barrier
2. G1采用的是SATB（snapshot-at-the-beginning）

CMS为了维持对象图变化才引入了post-write barrier 记录新引用关系的变化，这个维护成本很大，并且致命问题是，无法持续跟踪堆外根集变化，这样一来，remark阶段就需要重新扫描整个 GC Roots，新生代以及寄存器，too expensive，在处理大堆的时候很可能严重影响停顿时间；SATB跟CMS 增量barrier完全是另一套思路，关注的是引用关系的删除，使用的是pre-write barrier，在删除/变更 引用关系之前把旧值(old value)记录下来，对于在并发标记期间删除/变更的引用关系的旧值会记录下来，并且让引用指向的对象存活过此次GC，即使这个对象有可能并非存活，也只是多了些float garbage而已，在remark阶段也不需要重新扫描所有GC roots了，因为只要是新生成的对象，都视为alive。虽然两者remark阶段同为STW，但是实质上处理的事情和延时可能会有本质的区别，尤其是面对大堆的时候。

**2. CMS和G1启动GC的时机？**

- 对于CMS，配置-XX:CMSInitiatingOccupancyFraction = n即可，**注意这里的n表示垃圾对象在老年代的空间占比**
- 对于G1，配置-XX:InitiatingHeapOccupancyPercent = n，**表示垃圾对象在整个G1堆内存的空间占比（Mixed GC）**

**3. 什么时间会出现 Full GC？**

- 对于CMS垃圾回收器：
  - Concurrent-mode-failure：当 CMS GC 正进行时，此时有新的对象要进行老年代，但是老年代空间不足造成的；
  - Promotion-failed：当进行 Young GC 时，有部分新生代代对象仍然可用，但是S0或S1放不下，因此需要放到老年代，但此时老年代空间无法容纳这些对象。
- 对于G1垃圾回收器：
  - 如果Mixed GC实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行 GC，就会切换到 G1 之外的 Serial old GC 来收集整个 GC Heap（注意，回收区域包括 young、old、perm），所以对于正常工作的G1垃圾回收期是不能存在Full GC的，如果真出现了，估计就很悲剧了，毕竟单线程 + 大内存 + 整个堆，时间开销可想而知。

**4. Full GC、Magjor GC、Minor GC、Young GC 之间的关系？**

- Full GC == Major GC指的是对老年代/永久代的stop the world的GC
- Full GC的次数 = 老年代GC时STW的次数
- Full GC的时间 = 老年代GC时STW的总时间
- CMS 不等于 Full GC，我们可以看到 CMS 分为多个阶段，只有STW的阶段被计算到了Full GC的次数和时间，而和业务线程并发的 GC 的次数和时间则不被认为是Full GC
- Full GC本身不会先进行 Young GC，我们可以配置让Full GC之前先进行一次Young GC，因为老年代很多对象都会引用到新生代的对象，先进行一次Young GC可以提高老年代GC的速度。比如老年代使用CMS时，设置CMSScavengeBeforeRemark优化，让 CMS remark 之前先进行一次 Young GC

&nbsp;

## **3.5 ZGC**
- 着色指针 Colored Pointer：ZGC 利用指针的 64 位中的几位表示 Finalizable、Remapped、Marked1、Marked0（ZGC 仅支持 64 位平台），以标记该指向内存的存储状态。相当于在对象的指针上标注了对象的信息；在这个被指向的内存发生变化的时候（内存在整理时被移动），颜色就会发生变化
- 读屏障 Load Barrier：由于着色指针的存在，在程序运行时访问对象的时候，可以轻易知道对象在内存的存储状态（通过指针访问对象），若请求读的内存在被着色了。那么则会触发读屏障。读屏障会更新指针再返回结果，此过程有一定的耗费，从而达到与用户线程并发的效果
- ZGC 和 G1 一样将堆划分为 Region 来清理、移动，稍有不同的是 ZGC 中 Region 的大小是会动态变化的

### **3.5.1 过程**

- 初始停顿标记（STW）
- 并发标记
- 移动对象
- 修正指针

&nbsp;

# **四、类加载**

## **4.1 类加载过程**
加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载

- 加载：通过类的全限定名获得该类的二进制字节流，并将二进制字节流所代表的静态存储结构转化为方法区运行时存储结构，最后在内存中生成java.lang.Class对象
- 验证：
  - 文件格式验证：是否以魔数开头，主次版本号是否在当前虚拟机处理范围之内
  - 元数据验证：类的元数据信息进行语义检验，例如该类是否有父类，是否继承了不允许继承的类，或者抽象类是否实现了方法
  - 字节码验证：类的方法体进行校验，例如子类对象可以赋值给父类数据类型，相反则是不合法的
  - 符号引用验证：比如符号引用中通过字符串描述的全限定名是否能找到对应的类
- 准备：为类变量分配内存；设置类变量初始值；
  - 例如public static int value = 123，准备阶段value = 0，赋值123是在初始化阶段执行，但如果加了final关键字，说明存在ConstantValue属性，准备阶段value = 123
- 解析：将常量池中的符号引用替换为直接引用
  - 符号引用使用一组符号来描述所引用的目标，可以是任何形式的字面常量，定义在Class文件格式中（CONSTANT_Utf8_info等）
  - 直接引用可以是直接指向目标的指针、相对偏移量或则能间接定位到目标的句柄
- 初始化：初始化阶段即虚拟机执行类构造器<clinit>()方法的过程

## **4.2 类加载顺序**
```
父类  静态变量/静态代码块  （看二者在代码中的先后顺序）
子类  静态变量/静态代码块
父类  普通变量/普通代码块  （看二者在代码中的先后顺序）
父类  构造函数
子类  普通变量/普通代码块  （看二者在代码中的先后顺序）
子类  构造函数
```

## **4.3 类与类加载器**
**对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间**




# **五、引用类型**

- 强引用：垃圾回收器绝不会回收它，如Object obj = new Object()
- 软引用（SoftReference）：可有可无，内存空间足够，就不会回收它，如果内存空间不足了，就会回收；用来实现内存敏感的高速缓存，和引用队列（ReferenceQueue）联合使用
- 弱引用（WeakReference）：可有可无，弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期；在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。Weak引用对象会更容易、更快被GC回收
- 虚引用（PhantomReference）：用来跟踪对象被垃圾回收器回收的活动，虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列联合使用；回收器发现对象还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列（无法通过虚引用取得实例，不会对生存时间构成影响，垃圾回收时收到一个系统通知）


# **六、JVM参数调优&问题**

## **1.JVM CodeCache打满引起CPU Load偏高**
**背景：系统报出cpu load偏高，并且有业务反馈99先不稳**

**步骤：**
1. ps ux查看哪个进程导致CPU load值偏高，有可能是业务进程，也可能是其他系统进程（ps -ef | grep java）
2. 查看cpu占用线程top -H -p <pid>（步骤1所得进程id），可以看到pid进程下各个线程占用cpu情况，找出占用cpu最大的线程id
3. 转换线程id
  - 通过top命令看到的线程id是10进制的，后续通过jstack导出的线程堆栈是16进制的，使用printf “%x” threadId
4. 导出线程堆栈 jstack pid>out.txt
5. 发现是JIT编译器的C1编译线程，相关的是JVM CodeCache

**原因：**
1. JIT的C1和C2编译器将编译后的native code存储到CodeCache，在下次执行时直接执行native code，这样会大大加快执行速度。但，当CodeCache存储区域满之后，JIT就会停止编译，并且此过程是不可逆的，因此后续的代码不会进行这类优化，就导致服务器性能有所降低，也就是前面所说的99线不稳。
2. 另外一个方面，CodeCache这块的垃圾回收算法存在一些问题【相关资料6】，就导致CodeCache满之后，JIT的C1和C2线程占用大量CPU，导致机器load持续偏高，这也是Code Cache的一个BUG
