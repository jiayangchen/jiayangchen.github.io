---
layout: post
title: "【译】Redis Partitioning"
subtitle: ""
date: 2018-01-07
author: "ChenJY"
header-img: "img/code1.jpg"
catalog: true
tags: 
    - Redis
    - 翻译
---

> 翻译自 [Redis Partitioning](https://redis.io/topics/partitioning)

# 分片：如何数据分隔之后存放在多个 Redis 实例中

分片（Partitioning）是将你的数据分割之后存入多个 Redis 实例的过程，因此每个实例只包含所有 keys 的一个子集。本文的第一部分将向你介绍分片的概念，第二部分将向你展示 Redis 分片的备选方案。

## 为什么分片很有用
Redis 分片服务于两个最主要的目的：

* 它允许更大的数据库，使用许多计算机的内存总和。如果没有分片，你只能被限制在使用单一计算机的内存大小。

* 