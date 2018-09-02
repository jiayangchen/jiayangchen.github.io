---
layout: post
title: "Zookeeper 在构建系统中的作用"
subtitle: ""
date: 2018-08-25
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - Java Tech
    - 分布式系统
---

这篇主要聊一聊，Zookeeper 在业界构建系统时，一般承担什么样的功能，注册中心？分布式锁？亦或是 Name Serivce 等

## 什么是 Zookeeper 
Zookeeper 是 Yahoo! 基于 Google Chubby 论文的思想，结合自己的实践经验开发而来，2011 年成为 Apache 顶级项目，本质上是一个高可用的分布式协调服务（highly reliable distributed coordination），应用可以基于它实现数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Leader 选举、分布式锁等一系列服务。

## 应用场景
那 Zookeeper 的常见应用场景有哪些呢？这里列举几个典型应用。

### 数据发布订阅
Zookeeper 采用推拉结合的方式，客户端可以在 Zookeeper 上对感兴趣的节点注册 Watcher，而服务端借助 Watcher 机制可以方便地向客户端推送数据变更的事件通知，然后客户端会发起请求拉取最新数据。这样的特性有诸多应用场景，例如动态开关、监控、配置信息等，它们具有数据量小，时效性要求高的特点。**这里受限于 Zookeeper 本身的性能瓶颈，目前配置中心这种服务越来越倾向于去 zk 化，改用其他的实现方式。**

### 注册中心
例如 Dubbo 支持以 Zookeeper 作为服务注册中心，这就是一种动态 DNS 解析服务，借此服务的提供者和调用者间可以解耦，批次的更改可以通过注册中心做到无感知变更，需要做的只是替换服务的节点地址，再通过 Zookeeper 将变更推送到服务的调用方，刷新本地缓存的路由。

### 选举 Leader
这应该是最常见的使用方式了，应用接入 Zookeeper 实现选主功能来保证在网络分区的时候替换 Leader，达到高可用的目的。如何实现Leader 选举呢，这里推荐直接使用 Netflix 开源的 Curator 框架来进行，例如可以参考笔者同事的这篇博客 [Curator leader选举](http://yeming.me/2018/08/24/curatorLeader/)，如果想采取原生 API 的形式，那么可以根据多台机器同时注册节点仅有一台会成功的特性，让多台机器竞争创建一个临时节点，然后成功的那个当选为 Leader，其余机器注册该节点的 Watcher，来探测 Leader 的存活。一旦发现 Leader 挂了，那么其余机器立刻竞争选举 Leader。后者这种机制是很原始的，风险较大适合简单情况下的使用，推荐前者 Curator 框架。

### 心跳探测
常见的方式是通过 Ping 命令定时探测主机是否正常回应，来判断工作状态的。而借助 Zookeeper 的临时节点，也可以达到这种目的。不同的机器都在指定节点下创建临时子节点，不同的机器间根据临时节点来判断彼此间是否存活。

### 分布式锁
这也是 Zookeeper 的高频应用场景。网上资料非常多，这里不再赘述。其原理同样是基于竞争创建节点仅有一个会成功的特性，而释放锁则是基于临时节点的特性确保锁不会被永久持有。另外，锁的正确性还需要借助节点序号，这里同笔者之前发布的基于 Redis 的分布式锁原理基本是相近的，感兴趣可以看看[怎样做可靠的分布式锁，Redlock 真的可行么](https://chenjiayang.me/2018/08/05/how-to-do-distributed-lock/)。但是，建议使用成熟的 Zookeeper 操作框架来进行该功能的使用，不建议采用原生 API 自己实现的方式。

## 结论
总而言之，现在 Zookeeper 同 Redis 一样，在开发过程中的使用频率越来越高，了解常见的使用场景会加深对其的理解。但是 Zookeeper 并不是万能的组件，其饱受诟病的就是性能表现，Leader 一旦撑不住高流量就会崩溃；Leader 选举过程因为基于 Paxos 改进，因此效率不高；此外如果不借助成熟的框架例如 Curator，那么 Zookeeper 其实只暴露一堆 API 而已，至于怎么去实现可靠的选举、注册等功能，还是需要开发人员自己万般小心。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>


