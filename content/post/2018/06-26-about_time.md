---
title: "关于秒相关的时间"
date: 2018-06-26T21:26:15+08:00
lastmod: 2018-06-26T21:26:15+08:00
draft: false
keywords:
- "秒时间"
description: ""
tags:
- "计算机基础"
categories:
- "日常基础"
author: "imet"

mathjax: true
mathjaxEnableSingleDollar: true
mathjaxEnableAutoNumber: false
---

关于比秒更小的时间单位，CPU 时钟周期用什么度量，佛曰“一刹那”是多长时间呢。

<!--more-->

## 秒时间表格

| 名称      | 英文             | 大小       | 频率 | 备注                                      |
|-----------|------------------|------------|------|-------------------------------------------|
| 秒        | second (s)       | 1          | 1Hz  |                                           |
| 毫秒      | millisecond (ms) | $10^{-3}$  | 1KHz |                                           |
| 微秒      | microsecond (µs) | $10^{-6}$  | 1MHz |                                           |
| 纳秒      | nanosecond (ns)  | $10^{-9}$  | 1GHz | CPU 时钟周期                              |
| 皮秒      | picosecond (ps)  | $10^{-12}$ |      |                                           |
| 飞秒      | femtosecond (fs) | $10^{-15}$ |      |                                           |
| 阿秒      | attosecond (as)  | $10^{-18}$ |      | 目前实验上能测量的最小时间尺度            |
| 仄普托秒  | zeptosecond (zs) | $10^{-21}$ |      | 放射性原子核衰变中γ射线辐射的典型周期时间 |
| 遥刻托秒  | yoctosecond (ys) | $10^{-24}$ |      |                                           |
| planktime | plank time (tP)  | $10^{-44}$ |      | 普朗克時間                                |

## 关于时间的思考
1. 阿秒: 目前实验上能测量的最小时间尺度
2. 飞秒: 可见光的周期
3. 纳秒: CPU 时钟周期，主要是对 CPU 2.xGHz 感兴趣，才查了一下时间的最小单位，所以 CPU 脉冲电压是 0.3~0.4ns 纳秒左右
4. 印度《摩訶僧祇律》记载：“須臾者。二十念名一瞬頃。二十瞬名一彈指。二十彈指名一羅豫。二十羅豫名一須臾。日極長時有十八須臾，夜極短時有十二須臾。夜極長時有十八須臾，日極短時有十二須臾”:
   一刹那大概是 18 毫秒

## 参考
1. `https://en.wikipedia.org/wiki/Orders_of_magnitude_(time)`
2. `https://zh.wikipedia.org/wiki/%E6%95%B0%E9%87%8F%E7%BA%A7_(%E6%97%B6%E9%97%B4)`
3. https://zh.wikipedia.org/wiki/%E5%88%B9%E9%82%A3
