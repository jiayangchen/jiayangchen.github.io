---
layout: post
title: "The Chubby lock service for loosely-coupled distributed systems"
subtitle: "论文阅读系列"
date: 2018-07-20
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 论文阅读
    - 分布式系统
---

主要介绍了谷歌内部的分布式锁服务 Chubby，可以在分布式环境下提供粗粒度的锁服务，可用于大规模机器间实现同步、存储元数据或者拓扑结构等信息。谈到如何实现异步的一致性，Chubby 的解决方案是引入 Paxos，事实上作者也提到当时只要是实现了异步一致性的或多或少其内部实现都有 Paxos 算法的影子。作者强调这篇论文并不涉及新提出的算法，而只是聚焦于工程实现和后续优化提升。



