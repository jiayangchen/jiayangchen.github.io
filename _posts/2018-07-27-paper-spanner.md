---
layout: post
title: "Spanner: Google’s Globally-Distributed Database"
subtitle: "论文阅读系列"
date: 2018-07-27
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 论文阅读
    - 分布式系统
---

Spanner 是谷歌的可伸缩、多版本、全球分布、支持同步复制的数据库，它是第一个在全球范围内传递数据且保证外部一致的分布式事务的系统。论文叙述了它的架构、特征、许多设计的依据和一个新的可以暴露时钟不确定性的API。

## 简介

开篇叙述了 Spanner 的一些特性，在全球分布的 Paxos 状态机上进行 data sharding，利用复制保证整体可用性和地域局部性，客户端会自动在副本间进行failover，当数据数量或者服务器数量改变时，Spanner 会自动进行 data resharding，且它还支持在不同机器（甚至数据中心间）迁移数据以此来进行负载均衡或者应对突发的错误。

接着作者讲述了 Google 内部 Bigtable 和 Megastore 的一些问题，前者对于复杂可变的模式，或者需要大范围强一致性复制的场合不适用；后者的写吞吐量很差，即使是它的半关系型数据模型和对实施复制的支持很好，但是也非完美的选择。最后的结果是，Spanner 从一个与 Bigtable 相似的 KV 存储进化成了一个数据多版本的数据库。数据存储在模式化、半关系型的表中；数据有版本的区分，数据在 commit 的时候会为每个版本生成一个 timestamp，旧版本会受制于垃圾回收机制；同时应用也可以读旧版本的数据。Spanner 支持通用的事务，提供了基于 SQL 的查询语言。

接着作者概括了一些 Spanner 的特性：第一是应用可以自己细粒度的控制数据复制的配置项，包括哪个 datacenter 包含什么数据、数据离用户多远（控制读延迟）、副本间距多远（控制写延迟）、有几个副本（决定可用性和容灾等级），数据也可以在 datacenter 之间流动，以平衡资源使用。第二 Spanner 实现了两个分布式数据库的难点，（1）外部一致的读写操作 （2）在一个 timestamp 下全球一致的跨数据中心的读操作。这些特性使得 Spanner 可以在全球层面上，支持一致性备份、一致的 MapReduce 任务执行，和对 schema 原子更新操作，即使是事务正在进行中也可以。

怎么实现的呢？作者也说了得益于 Spanner 为事务分配一个全球意义的 commit timestamp，即使事务是分布式的。这些 timestamp 可以反映串行顺序，另外串行顺序又能支持外部一致：例如事务 T1 在 T2 start 之前 commit，那么 T1 的 commit timestamp 就会比 T2 的小。这个 timestamp 是由一个新的 TrueTime API 提供的，这个 API 能暴露时钟的不确定性，如果这个不确定性大了，那么 Spanner 就会慢下来等待不确定性变小，Google 的实现通过使用现代时钟参考（例如原子钟或者GPS）能保持这种不确定性小于 10ms。

## 实现机制

首先作者介绍了三个概念，分别是 *directory*、*universe*、*zone*，其中 *directory* 是一个抽象概念，来管理 replication 和 locality，并且也是数据移动的单元；*universe* 是一个 Spanner 的部署；Spanner 被组织成许多 *zone*，每个 *zone* 像一个部署完成 Bigtable 服务器组，*zone* 也是管理部署的单元，当新的 datacenter 开始服务时，它可以增加，当有 datacenter 关闭时它也可以被移除，*zone* 同样也是物理隔离的单元，一个 datacenter 中可能存在多个 *zone* ，例如有应用数据需要在同个 datacenter 中划分存储在多个 server 上时。

![](https://upload-images.jianshu.io/upload_images/1752522-8d4d85a54036697f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中每个 zone 有一个 zonemaster 和 成千上万 spanserver，前者向 spanserver 分配数据，后者则向客户端提供数据，location proxy 则是用来向客户端定位 spanserver 的，universemaster 是一个控制台显示状态，placement driver 定时和 spanserver 沟通发现需要被移动的数据或者更新复制约束或者是负载均衡。

![](https://upload-images.jianshu.io/upload_images/1752522-b99c0dffe8361eb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是 spanserver 的软件架构，其中每个 spanserver 负责一百至一千个数据实例 *tablet*，它和 Bigtable 中的概念有相似也有不同，它通过如果形势组织数据：

> (key:string, timestamp:int 64) -> string

timestamp 在 key 上而不是在 data 上（Bigtable 中数据具有不同的timestamp），这也是为什么 Spanner 更像多版本数据库而不是KV数据库的原因。每个 Tablet 的状态被存储在一系列类似 B-Tree 结果的文件和一个 write-ahead（写前操作） 日志中。为了支持复制，每个 spanserver 在自己 tablet 的顶层实现了一个 Paxos 状态机，我们的 Paxos 实现通过基于时间的租约而支持长时间存活的 leader。副本的集合被称为一个 Paxos Group，写操作必须在 leader 上初始化 Paxos 协议，读操作可以在任意一个足够新的副本上进行。每个副本 leader 上 spanserver 实现了 lock table 来控制并发，它把 key 的范围映射到锁状态上，对于需要同步的操作例如事务读需要获取锁表中的锁，其他操作不需要。

为了支持分布式事务，每个 spanserver 实现了事务管理器，它会实现一个 participant leader，当事务中仅有一个 Paxos Group 时可以忽略事务管理器，因为依靠锁表和 Paxos 就可以支持事务，如果涉及多个 Paxos Group，那么这些 Group 的 leader 协商进行 Two-phase commit，其中一个 Group 被选出担任协调者（coordinator），那么这个 Group 中的 participant leader 就会变成 coordinator leader，Group 中的其他副本变成 coordinator slaves 用于容灾。

![](https://upload-images.jianshu.io/upload_images/1752522-192ba58099b34f11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*directory* 里是一段连续的、具有公共前缀的 Keys，数据在不同的 Paxos Group 间是一个一个 directory 移动的，Movedir 是后台移动数据的任务，也可以用来添加和删除副本，它会开启一个事务用于转移数据的最后一部分，然后借助事务更新两个 Paxos Group 的元数据。事实上，Spanner 会将大的 directory 切分成 segment，segment也会被保存在不同的分组上，转移时是转移 segment。

 这节最后作者介绍了 Spanner 的数据模型，模式化的半关系型表，由 Megastore 支持，类似 SQL 的查询也没有讲述细节，这部分感兴趣的可以看 [TiDB](https://www.pingcap.com/) 的具体实现。

## TrueTime API

 ![](https://upload-images.jianshu.io/upload_images/1752522-d1557f77c3323255.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于一个调用 tt=TT.now()，有 tt.earliest ≤ tabs(enow)≤tt.latest，其中， enow 是调用的事件。在底层，TrueTime API 使用的时间是 GPS 和原子钟。TrueTime 是由每个 datacenter 上面的许多 time master 机器和每台机器上的一个 timeslave daemon 来共同实现的。大多数 master 都有具备专用天线的 GPS 接收器，这些 master 在物理上是相互隔离的，这样可以减少天线失效、电磁干扰和电子欺骗的影响。剩余的 master (我们称为 Armageddon master) 则配备了原子钟。所有 master 的时间 参考值都会进行彼此校对。每个 master 也会交叉检查时间参考值和本地时间的比值，如果二者差别太大，就会把自己驱逐出去。每个 daemon 会从许多 master（可能是附近的也可能是很远的 datacenter的 ） 中收集投票，获得时间参考值，从而减少误差，这里还会涉及到对于 liar 的判断和剔除。

## 并发控制

![](https://upload-images.jianshu.io/upload_images/1752522-91fa02b85deaedff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Spanner 支持上述的操作，其中，一个只读事务必须事先被声明不会包含任何写操作，在一个只读事务中的读操作，在执行时会采用一个系统选择的时间戳，不包含锁机制，因此，后面到达的写操作不会被阻塞，可以到任何足够新的副本上去执行。一个快照读操作，是针对历史数据的读取，执行过程中，不需要锁机制。对于只读事务和快照读而言，当一个服务器失效的时候，客户端就可以使用同样的时间戳和当前的读位置，在另外一个服务器上继续执行读操作。

**Paxos 租约与 Chubby 类似，都是大多数投票加续租的模式，不再多讲了，可以看我关于 Chubby 论文的笔记。**

事务读和写采用 Two-phase locking，当所有的锁都获得后且在任意锁被释放前，可以给事务分配 timestamp。对于一个给定的事务，Spanner 会为事务分配 timestamp，这个 timestamp 是 Paxos 分配给 Paxos write 操作的，它代表了事务提交的时间。

Spanner 的事务在内部依赖于一些单调性，在外部一致性的表现上，如果一个事务 T2 在事务 T1 提交以后开始执行， 那么，事务 T2 的时间戳一定比事务 T1 的时间戳大。

### 读操作

对于在某个 timestamp 下进行读操作，每个副本都会跟踪记录一个值，这个值被称为安全时间 tsafe，它是一个副本最近更新后的最大时间戳。如果一个读操作的时间戳是 t，当满足 t <= tsafe 时， 这个副本就可以被这个读操作读取。

### 读写事务和只读事务细节

**这部分较为复杂，概括来讲还是通过 timestamp 保持单调性，引入失效等待机制保证执行顺序和条件阻塞，内容繁多请直接看论文吧**

### schema 原子更新

一个 Spanner schema 变更事务通常是一个标准事务的、非阻塞的变种。首先，它会显式地分配一个未来的 timestamp，这个 timestamp 会在准备阶段进行注册。由此，跨越几千个服务器的模式变更，可以在不打扰其他并发活动的前提下完成。

## 总结

论文中指出Spanner 汇集和扩展了两个研究社区的概念：一个是数据库研究社区，包括熟悉易用的半关系接口，事务和基于 SQL 的查询语言；另一个是系统研究社区，包括可扩展性，自动分区，容错，一致性复制，外部一致性和大范围分布。设计中一个亮点特性就是 TrueTime API，以更加强壮的时间语义来构建分布式系统。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>


