---
title: "Nginx"
date: 2019-04-06T11:30:47+08:00
lastmod: 2019-04-06T11:30:47+08:00
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
前言
Nginx是一款轻量级的Web服务器、反向代理服务器，由于它的内存占用少，启动极快，高并发能力强，在互联网项目中广泛应用

![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20190406113629.png)
Ngnix中的进程分为两类，一类是master进程，一类是worker进程。 　　
其中master进程使用来管理监控控制其下边的worker进程的主进程，这个进程由root发起。其中原因是http这个服务需要启用80端口，而只有root才有权限启用80端口。





参考文章
https://www.jianshu.com/p/5eab0f83e3b4
https://segmentfault.com/a/1190000011348130
