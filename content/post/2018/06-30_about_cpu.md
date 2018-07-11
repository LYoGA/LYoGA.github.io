---
title: "关于CPU的一些知识"
date: 2018-06-30T23:47:39+08:00
lastmod: 2018-06-30T23:47:39+08:00
draft: false
keywords:
- "CPU"
- "Cache"
description: "关于CPU的一些知识"
tags:
- "计算机基础"
categories:
- "日常基础"
author: "imet"

---

关于 CPU 运行的有多快呢，CPU 执行一条命令需要多长时间，CPU，L1，L2，内存，硬盘时间上的差距呢

<!--more-->

## CPU 内存等硬件时间表格

硬件                             | 时间（ns）    | 转换时间
---------------------------------|---------------|---------------------
cpu                              | 0.38ns        |
L1                               | 0.5ns         |
分支预测错误                     | 5ns           |
L2                               | 7ns           |
互斥锁加锁                       | 25ns          |
内存寻址                         | 100ns         |
上下文切换                       | 1500ns        | 1.5µs
zippy压缩 1KB 数据               | 3,000ns       | 3µs
1Gps网络发送 2KB 数据            | 20,000ns      | 20µs
SSD 随机读                       | 150,000ns     | 150µs
从内存中读取 1MB 的连续数据      | 250,000ns     | 250us
同一个数据中心跑一个来回         | 500,000ns     | 0.5ms
从 SSD 读取 1MB 的顺序数据       | 1,000,000ns   | 1ms
磁盘寻址时间为                   | 10,000,000ns  | 10ms
从世界上不同城市网络上走一个来回 | 150,000,000ns | 150ms (参考ping报文)

## mac cpu，L1，L2

命令行查看 cpu 相关信息

```bash
$ sysctl -n machdep.cpu.brand_string
Intel(R) Core(TM) i7-4770HQ CPU @ 2.20GHz

$ system_profiler SPHardwareDataType
Hardware:

    Hardware Overview:

      Model Name: MacBook Pro
      Model Identifier: MacBookPro11,4
      Processor Name: Intel Core i7
      Processor Speed: 2.2 GHz
      Number of Processors: 1
      Total Number of Cores: 4
      L2 Cache (per Core): 256 KB
      L3 Cache: 6 MB
      Memory: 16 GB
      Boot ROM Version: MBP114.0183.B00
      SMC Version (system): 2.29f24
      Serial Number (system): xxxxxxxx
      Hardware UUID: xxxxxxxxxxxx
```

## 相关问题

### 1. 为什么 CPU 时钟频率不能再提高 ？

可以参考[链接][9]和[知乎上的链接][11]，了解一下 CPU 为啥时钟频率不能再提高，总体来说是受到现在制成工艺的限制

### 2. L1，L2，L3 的结构

顺便看一下 L1, L2, L3 的结构:
![cpu_cache](http://pb56ttwyu.bkt.clouddn.com/blog/img/20180630232448_cpu_cache.png)

可以参考[链接][10]，了解缓存寻址时间

### 3. CPU 常说4核8线程是啥？

Intel CPU 通过超线程，实现了一个 CPU 核心跑 2 个线程，来模拟 2 个 CPU 核心。
- 物理核数量 = CPU 数(机子上装的 CPU 的数量) * 每个 CPU 的核心数
- 上图也描述了一个 CPU core 里面 2个线程

所以一般 4 核就是 8 CPU 线程来最大化 CPU 的利用率，可以参考[超线程][13]，[多线程][14]相关的介绍

### 4. CPU 运算速度用什么来衡量？

IPS: Instructions Per Second，每秒指令次数
MIPS: Million Instructions Per Second，每秒百万指令次数

MIPS 是衡量 CPU 运算速度的一个标准

看来要补 cs 课程了


## 参考
1. [让 CPU 告诉你硬盘和网络到底有多慢](http://cizixs.com/2017/01/03/how-slow-is-disk-and-network)
2. [What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf)
3. [Latency numbers every programmer should know](https://gist.github.com/hellerbarde/2843375)
4. [Latency Numbers(by year) Every Programmer Should Know](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html)
5. [Getting Physical With Memory](https://manybutfinite.com/post/getting-physical-with-memory/)
6. [how long does it take to make context](http://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)
7. [How do I identify which CPU a MacBook uses?](https://apple.stackexchange.com/questions/238777/how-do-i-identify-which-cpu-a-macbook-uses)
8. [i7-4770HQ](https://ark.intel.com/products/83505/Intel-Core-i7-4770HQ-Processor-6M-Cache-up-to-3_40-GHz)
9. [Why haven't CPU clock speeds increased in the last 5 years?](https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years)
10. [理解 CPU Cache](http://wsfdl.com/linux/2016/06/11/%E7%90%86%E8%A7%A3CPU%E7%9A%84cache.html)
11. [为什么 CPU 主频很难超过 4GHz？](https://www.zhihu.com/question/32096371)
12. [每秒指令](https://zh.wikipedia.org/zh-cn/%E6%AF%8F%E7%A7%92%E6%8C%87%E4%BB%A4)
13. [Intel 超线程](https://www.intel.com/content/www/us/en/architecture-and-technology/hyper-threading/hyper-threading-technology.html)
14. [Multithreading (computer architecture)](https://en.wikipedia.org/wiki/Multithreading_%28computer_architecture%29)
15. [超线程](https://zh.wikipedia.org/zh-cn/%E8%B6%85%E5%9F%B7%E8%A1%8C%E7%B7%92)
16. [认识cpu、核与线程](http://www.cnblogs.com/-new/p/7234332.html)

[9]: https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years
[10]: http://wsfdl.com/linux/2016/06/11/%E7%90%86%E8%A7%A3CPU%E7%9A%84cache.html
[11]: https://www.zhihu.com/question/32096371
[13]: https://www.intel.com/content/www/us/en/architecture-and-technology/hyper-threading/hyper-threading-technology.html
[14]: https://en.wikipedia.org/wiki/Multithreading_%28computer_architecture%29
