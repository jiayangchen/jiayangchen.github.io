---
layout: post
title: "Zookeeper 在构建系统中的作用"
subtitle: ""
date: 2018-07-27
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - Java Tech
    - 分布式系统
---

这篇主要聊一聊，Zookeeper 在业界构建系统时，一般承担什么样的功能，注册中心？分布式锁？

## 什么是 Zookeeper 

Zookeeper 是 Yahoo! 基于 Google Chubby 论文的思想，结合自己的实践经验开发而来，2011 年成为 Apache 顶级项目，本质上是一个分布式协调服务，应用可以基于它实现数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Leader 选举、分布式锁等一系列服务。

