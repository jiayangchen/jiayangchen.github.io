---
layout: post
title: "Cassandra - A Decentralized Structured Storage System"
subtitle: "论文阅读系列"
date: 2018-09-02
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 论文阅读
    - 分布式系统
---

## Intro & Related Work
Cassandra 来自于 Facebook 的分布式存储系统，目的是取得可拓展性和高可用性，一开始主要为了 Inbox Search 这个功能设计，该功能要求系统能处理高频的写入吞吐量，为了减少延迟也要求跨地域部署。相关工作这部分简述了几个代表性的分布式存储系统实现，包括 GFS，Dynamo，Bayou，Ficus 等，讲了他们在架构设计、副本设计、冲突解决等因素上的权衡选择。

## 数据模型
Cassandra 里 table 每行都有唯一 key，string 类型无长度限制，一般16~36字节间，对每个副本来说对某一行的操作不论设计多少 column 都是原子的，column 被设计成 column family 的形式，很像 BigTable，但是有两种类型，一种 Simple 的一种 Super，Super 的就是 family 嵌套 family。系统可以允许 columns 按 name 或者时间排序。

定位 column 的方式很简答，column_family：column 或者对于 super 来说就是 column_family：super_column：column。

## 系统架构
限于篇幅只介绍 Cassandra 使用的 Partitioning、Replication、membership、failure handling 和 scaling。

对写操作将请求路由至所有副本，等待大多数副本回应则成功；对读请求则根据客户端的策略是路由到最近的副本还是路由到所有副本等到多数回应，前者不在意一致性，后者在意一致性。

### Partitioning
数据分区使用一致性哈希，但是做了顺序保证优化，目的是为了分担流量大的节点的压力，将流量低的节点 moving on。

### Replication
这里有个协调者节点，负责一个 range 里的 key，Cassandra 为客户端提供了多种数据复制的选择，例如“Rack Aware”、“Datacenter Aware”等，Cassandra 使用 Zookeeper 来为 nodes 选举 leader，leader 告诉节点你们负责的 key 的 range 是多少，这部分数据也在 node 本地缓存，这部分设计和 Dynamo 蛮像的。

### Membership
这部分使用高效的 anti-entropy Gossip 协议。失效探测使用 value 来表示节点存活性的怀疑程度而不是单纯的 true or false。

### Local Persistence
依赖本地文件系统做数据持久化，对内存数据的写操作先写 commit log，因为 commit log 顺序写所以性能很高，一旦内存数据到达阈值就 dump 到 disk，为了高效的查找也会为每行 key 生成索引。当磁盘小文件很多时合并进程会后台启动将其合并成一个大文件，很像 BigTable。

查询操作先查内存，没命中查磁盘，从新文件开始查到旧文件。为了避免查询没 key 的文件，这里使用布隆过滤器（可以看笔者这篇讲布隆过滤器的文章），对于 column 也会有索引可以直接跳转避免 scan 很多 column。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>