---
layout: post
title: "java 内存模型"
subtitle: ""
date: 2017-04-14
author: "ChenJY"
header-img: "img/drive.jpg"
catalog: true
tags: 
    - 深入理解Java虚拟机
---

最近在看周志明的《深入理解Java虚拟机》，将其中的内容总结下来写几篇总结好了。

Java虚拟机在执行Java程序的时候会把它所管理的内存分为若干个不同的数据区域，如下图：
![image](http://o9oomuync.bkt.clouddn.com/Java%20%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA.png)

### 程序计数器
程序计数器所需内存较小，可看作是指向当前线程所执行的字节码的行号指示器，字节码解释器工作时通过改变PC的值选择下一条字节码指令。
每个线程有自己独立的程序计数器，这样在多线程切换的情况下各线程的PC互不影响，独立储存。
如果线程正在执行Java方法，则程序计数器记录的是当前字节码指令的地址，若是Native方法，则为空。

### Java虚拟机栈


