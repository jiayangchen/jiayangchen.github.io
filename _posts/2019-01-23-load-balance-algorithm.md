---
layout: post
title: "常见的负载均衡算法总结"
subtitle: "单机压力过大促使架构向集群模式转变，那么对于请求必定需要一个分发器进行任务分配和流量权衡"
date: 2019-01-23
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java Tech
    - 算法
---

单机压力过大促使架构向集群模式转变，那么对于请求必定需要一个分发器进行任务分配和流量权衡，业界一般习惯于叫“负载均衡器”，但其实它的作用不仅仅在于平衡机器间的压力，还在于如何更智能的选择任务分发策略，并在一定时候提供降级等保护措施，总之对负载均衡器的理解不能局限在表面意思上。

本文谈谈常见的负载均衡算法，也是初级程序员面试中大概率出现的问题。

### 轮询
意思很简单，对台机器组成的集群，前置的负载均衡器就将进入的流量按照次序依次发往各台机器，并不关心每台机器的实际压力情况如何，算法较为简单粗暴

### 权重轮询
相比于直来直去的轮询，权重轮询或者叫加权轮询即在选择分发的时候考虑机器的压力情况，压力小的多分发点流量，压力大的少分配一些，最终使集群达到稳定的状态。

这里的权重既可以根据服务器的硬件配置高低来进行初始化配置，也可以采用动态的方式在服务器运行期间，根据监控数据来实时调整，后者显然更契合复杂的场景，但同时实现难度也会更高。

### 贪心最优
有的时候，我们纯粹可以将客户端的流量发往目前延时最低、或者压力最小、或者配置最高的机器已达到短期内提升服务质量的目的，但显然长期生产环境中这不是个好方法，但不失为一个补充措施。为了实现这种算法，需要负载均衡器能收集集群机器的监控数据、响应时长等信息来进行分析。

### 哈希
哈希是常见的分发、散列算法，好的哈希算法可以很好地平摊流量，常见的可以根据 ip 哈希或者请求 ID 哈希等。ip 哈希可将同一个 ip 地址的请求分配给同一个服务器进行处理，可以解决 session 粘滞问题。ID 哈希中的 ID 一般是临时性数据的 ID（如 session id、token 等）。例如，网站登录用 session id hash 同样可以实现同一个会话期间，用户每次都是访问到同一台服务器的目的。

### License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>