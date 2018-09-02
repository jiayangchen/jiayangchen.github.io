---
layout: post
title: "Megastore: Providing Scalable, Highly Available Storage for Interactive Services"
subtitle: "论文阅读系列"
date: 2018-08-11
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 论文阅读
    - 分布式系统
---

## 摘要
Megastore 是 Google 为了满足联系愈加紧密的 online service 而研发的存储服务，它结合了 NoSQL 的可拓展性和传统 RDBMS 的易用性，且在此基础上提供了强一致性和高可用性保障。他们提供了完全序列化的 ACID 语义和良好的数据分区，这种分区借助网络进行跨地域地副本同步，保证延迟在可接受的范围内且提供无缝的故障转移。论文介绍了 Megastore 的语义和复制算法，还包括一些使用 Megastore 支持 Google 产品大规模部署的经验。

## 1 简介
这节作者列举了目前 online service 要求存储服务具备的一些要求：高可拓展、闭环服务高速迭代、低延迟、一致性、高可用。然后说明传统的 RDBMS 虽然方便但是不方便拓展；NoSQL 虽然拓展性好，但是 API 简陋、没有一致性模型、延迟高等问题，最后引出 Megastore。

值得一提的是，文章提到Megastore 采用 Paxos 来复制主要用户数据。

## 2 高可用和拓展性
两种方法：

**对可用性，**他们实现了一种同步的、可容错的 log replicator 来优化长距离连接

**对拓展性，**他们提供将数据分区存储在许多小数据中心里，且每个数据中心的 replicated log 存储在一个 per-replica 的 NoSQL 数据中心中。

### 2.1 复制
作者提出仅仅在数据中心里的不同 host 上复制数据其实是不够的，考虑到机房、电力等其他故障时，因此他们决定在地理上进行大范围复制。

作者评估了三种复制模型：异步主从（丢数据风险和需要一致性协议保证 mastership）、同步主从（不丢数据但需要外部failure探测）、优化复制（一个replica group每个都能接收改变，异步传递变化给整个group，拓展性低延迟非常好，但是不可能支持事务）。

他们最后引入 Paxos 一致性协议，不需要一个特定的 master，但是考虑到地理范围大延迟高，需要引入 multiple replicated logs，每个负责它自己的数据分区。

### 2.2 数据分区和局部性

为了提高吞吐量和本地使用率，引入 Entity Groups 概念。数据分区存储在一系列 Entity Groups 中，每个都是独立的且在大范围地域上进行同步复制。一个 Group 中的 Entities 通过一条 ACID 事务进行变更（通过 Paxos 进行日志复制）。跨 Group 的操作通过两阶段提交，Group 间的异步交流通过 Queue，外加一些本地索引和全局索引。示意图如下，我个人觉得画得比较清晰：

![](https://pic4.zhimg.com/80/v2-f88c650d0feaa52a91ff49a0355a5bd2_hd.jpg)

在物理部署上他们使用 BigTable 作为一个数据中心的拓展容错存储，通过将操作分散至多行来支持随机读写吞吐。让应用自身控制数据安放位置来减小延迟提高吞吐量。

## 3 Megastore 预览

### 3.1 数据模型

Megastore 中的 table 有 root table 和 child table 之分，每个 child table 必须声明一个具有区分意义的外键指向 root table。

### 3.2 事务和并发控制
每个 Entity Group 都像一个小型的数据库提供 ACID 语义，一个事务在对 group

修改时先写日志（write-ahead log），然后修改数据。他们使用 BigTable 提高的时间戳特性从而实现了 MVCC 的多版本并发控制，当事务中的变更提交之后，值随着时间戳写入，然后读操作带上最后一次被完全采纳的事务时间戳来避免读到历史数据，且读写操作在事务持续期间可以彼此隔离。Megastore 提供 current、snapshot 和 inconsistent 读操作，其中 current 会等待所有事务日志提交完成且数据变更也完成之后读取；snapshot 则是只从最后一次事务完全提交处读取，忽略有些日志提交了但数据尚未变更的情况；inconsistent 则是只读最新值，忽略日志状态。

一个完整的事务流程:

![](https://pic3.zhimg.com/80/v2-2adb990f59b06879e3c404e6494d40b7_hd.jpg)


## 4 复制
Megastore 提供对数据单一的、一致的 view，读写操作都可以在任意副本上进行。其后作者谈了下 Paxos 算法和如何改进使之适应 Megastore 的需要。

### 4.4 Megastore 的做法

他们实现了一个协调者服务来跟踪那些所有副本均接受了 Paxos write 的 Entity Group 来支持 fast read 操作，Coordinator 保存在当前数据中心中哪些副本的数据是最新的，如果是这样的副本，就不必使用 Paxos 来得到最新数据，直接本地读取就可以。通过在 Entity Group 中加入 Leader 来支持快速写服务。最后，如果协调者服务失效了，写操作会被阻塞，这也就要求有一个机制检测协调者是否失效，Megastore 使用 Chubby Lock Service 检测失效。

## 5 结论
论文介绍了可拓展、高可用的存储服务 Megastore，使用 Paxos 来大范围同步复制数据，对每个操作提供轻量级和快速 failover，使用 BigTable 作为可伸缩的数据存储，且加入队列、索引和 ACID 事务。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>