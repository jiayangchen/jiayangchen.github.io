---
layout: post
title: "分析伪共享（False Sharing）产生原因"
subtitle: "什么是伪共享呢（False Sharing）呢，讲清楚伪共享出现的原因，我们要先理清楚高速缓存和 MESI 缓存一致性协议"
date: 2019-01-26
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 操作系统
---

之前读了一篇美团点评技术博客 2016 年发表的文章：[高性能队列——Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html "高性能队列——Disruptor")，里面提到了 `ArrayBlockingQueue`会因为加锁和伪共享等出现严重的性能问题。

什么是伪共享呢`（False Sharing）`呢，讲清楚伪共享出现的原因，我们要先理清楚高速缓存和`MESI`缓存一致性协议。

### Cache Memory

我们都知道 `CPU` 和主内存之间的运算速度是差异巨大的，在现今的 `SMP（Symmetric Multiprocessor）System` 中，会在 CPU 和主存间设置三级高速缓存，`L1`、`L2` 和 `L3`，读取顺序由先到后。实际上 `Cache` 的设计是经历过变更的，`Intel `和 `AMD` 的实现细节都不尽相同，本文你可以简单理解为，`L1 Cache `分为指令缓存和数据缓存两种，`L2 Cache `只存储数据，`L1` 和 `L2` 都是每个核心都有，而 `L3` 被多核共享。

### MESI

那么问题来了，多核`CPU`的情况下有多个 L1 和 L2 缓存，如何保证缓存内部数据的一致,不让系统数据混乱。这里就引出了一个一致性的协议`MESI`。

`MESI`规定了一个`cache line`存在四种状态：`Modified`、`Exclusive`、`Shared` 和` Invalid`，这有点像状态机的转换，理清全部的状态较复杂，我们关注简单的：

1. `Modified`：该缓存行只被缓存在该`CPU`的缓存中，并且是被修改过的，即与主存中的数据不一致，该缓存行中的内存需要在未来的某个时间点（允许其它CPU读取请主存中相应内存之前）写回主存。当被写回主存之后，该缓存行的状态会变成`Exclusive`状态。

2. `Exclusive`：该缓存行只被缓存在该`CPU`的缓存中，它是未被修改过的，与主存中数据一致。该状态可以在任何时刻当有其它`CPU`读取该内存时变成`Shared`状态。同样地，当CPU修改该缓存行中内容时，该状态可以变成`Modified`状态。

3. `Shared`：该状态意味着该缓存行可能被多个`CPU`缓存，并且各个缓存中的数据与主存数据一致，当有一个`CPU`修改该缓存行中，其它`CPU`中该缓存行可以被作废。

4. `Invalid`：该缓存是无效的（可能有其它`CPU`修改了该缓存行）。

### False Sharing

了解了这些之后，我们再来看伪共享问题，为什么它会影响到性能呢？

![](http://ww1.sinaimg.cn/large/c3beb895ly1fzk3usakk5g20eg0d1q2z.gif)

上图中`thread0`位于`core0`，而`thread1`位于`core1`，二者均想更新彼此独立的两个变量，但是由于两个变量位于同一个`cache line`中，此时可知的是两个`cache line`的状态应该都是`Shared`，而对于`cache line`的操作`core`间必须争夺主导权`（ownership）`，如果`core0`抢到了，`thread0`因此去更新`cache line`，会导致`core1`中的`cache line`状态变为`Invalid`，随后`thread1`去更新时必须通知`core0`将`cache line`刷回主存，然后它再从主从中`load`该`cache line`进高速缓存之后再进行修改，但令人抓狂的是，该修改又会使得`core0`的`cache line`失效，重复上演历史，从而高速缓存并未起到应有的作用，反而影响了性能。

### License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>














