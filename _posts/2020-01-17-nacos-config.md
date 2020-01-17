---
layout: post
title: "图文解析 Nacos 配置中心的实现"
subtitle: "本文不会贴太多源码，基本靠图片和文字叙述"
date: 2020-01-17
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java Tech
    - 架构设计
---

> 本文不会贴太多源码，基本靠图片和文字叙述

> 全文共 2582 字，预计阅读时间 12 分钟

- [什么是 Nacos](#什么是-nacos)
- [配置中心的架构](#配置中心的架构)
- [Nacos 使用示例](#nacos-使用示例)
    - [官方代码示例](#官方代码示例)
    - [Properties 解读](#properties-解读)
    - [配置项的层级设计](#配置项的层级设计)
- [Nacos 客户端解析](#nacos-客户端解析)
    - [获取配置](#获取配置)
    - [注册监听器](#注册监听器)
    - [配置长轮询](#配置长轮询)
- [Nacos 服务端解析](#nacos-服务端解析)
    - [配置 Dump](#配置-dump)
    - [配置注册](#配置注册)
    - [处理长轮询](#处理长轮询)
- [全文总结](#全文总结)

## 什么是 Nacos

`Nacos` 是阿里发起的开源项目，地址：[https://github.com/alibaba/nacos](https://github.com/alibaba/nacos)。`Nacos` 主要提供两种服务，一是配置中心，支持配置注册、变更下发、层级管理等，意义是不停机就可以动态刷新服务内部的配置项；二是作为命名服务，提供服务的注册和发现功能，通常用于在 `RPC` 框架的 `Client` 和 `Server` 中间充当媒介，还附带有健康监测、负载均衡等功能。

本文聚焦于 `Nacos` 的第一块功能，即配置中心的实现。先叙述一个配置中心通常需要哪些组成部分，再结合 `Nacos 1.1.4` 的源码，探究一下这些设计是如何反映在源码上的。

## 配置中心的架构

配置中心本身并不复杂，前提是你先将 `CAP` 的取舍问题晾在一边的话。配置中心最基础的功能就是存储一个键值对，用户发布一个配置（`configKey`），然后客户端获取这个配置项（`configValue`）；进阶的功能就是当某个配置项发生变更时，将变更告知客户端刷新旧值。

下方的架构图，简要描述了一个配置中心的大致架构，用户可以通过管理平台发布配置，通过 `HTTP` 调用将配置注册到服务端，服务端将之保存在 `MySQL` 等持久化存储引擎中；用户通过客户端 `SDK` 访问服务端的配置，同时建立 `HTTP` 的长轮询监听配置项变更，同时为了减轻服务端压力和保证容灾特性，配置项拉取到客户端之后会保存一份快照在本地文件中，SDK 优先读取文件里的内容。

这里省略了许多细节问题，例如配置分层设计，权限校验，客户端长轮询的间隔设置，服务端每次查询都需要访问 `MySQL` 么，配置变更是主动推送还是等定时轮询触发等，还有就是运维高可用方面的工作（私以为这个是配置中心的精华），例如节点跨地域部署，网络分区时配置如何保证可写可推送变更等。**真正实现一个高质量的配置中心，还是需要长时间打磨的。**

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579144438990-55f32e17-bd04-448a-8939-4c5a332e7a25.png#align=left&display=inline&height=364&name=image.png&originHeight=728&originWidth=1258&size=73238&status=done&style=stroke&width=629)

## Nacos 使用示例

> 下文涉及的源码均基于 Nacos 1.1.4 版本

### 官方代码示例

先看一下官方文档中对于 `Nacos` 的 `API` 使用的示例代码，第一步是传递配置，新建 `ConfigService` 实例，第二步可以通过相应的接口获取配置和注册配置监听器。使用方式非常简单易懂，不再赘述。

```java
try {
    // 传递配置
	String serverAddr = "{serverAddr}";
	String dataId = "{dataId}";
	String group = "{group}";
	Properties properties = new Properties();
	properties.put("serverAddr", serverAddr);
    
    // 新建 configService
	ConfigService configService = NacosFactory.createConfigService(properties);
	String content = configService.getConfig(dataId, group, 5000);
	System.out.println(content);
    
    // 注册监听器
    configService.addListener(dataId, group, new Listener() {
	@Override
	public void receiveConfigInfo(String configInfo) {
		System.out.println("recieve1:" + configInfo);
	}
	@Override
	public Executor getExecutor() {
		return null;
	}
});
} catch (NacosException e) {
    // TODO 
    -generated catch block
    e.printStackTrace();
}
```

### Properties 解读

`serverAddr` 传递的是配置中心服务端的地址列表，被内部名为 `ServerListManager` 的类解析成地址列表进行管理，进行 `HTTP` 调用时会从中选择存活的机器拼接成 `URL` 完成调用，一旦在调用时该地址抛异常，则客户端会有一些处理措施，例如转换下次选择的节点等。值得注意的是，通常在实践中不会采取这种硬编码的方式，可以将其配置在 `Zookeeper` 或者注册发现中心上，在启动时动态拉取。

### 配置项的层级设计

`Nacos` 官方给出了这样的设计图：

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579156674829-df2f915b-0661-4824-9077-1eaf0bccbbd5.png#align=left&display=inline&height=291&name=image.png&originHeight=582&originWidth=688&size=94526&status=done&style=stroke&width=344)

`dataId` 可以理解为用户自定义的配置健，`group` 可以理解为配置分组名称，这个属于配置层级设计的概念。简单来说，配置中心会通过层次设计，来支持不同的分区，以此区分不同的环境、不同的分组、甚至不同的开发者，满足在开发过程中灰度发布、测试等需求。因此怎样设计都可以，只要有含义就好，例如下图也不是不可以。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579157390667-544e35fb-285f-48c3-a720-0c8edb2cbca6.png#align=left&display=inline&height=448&name=image.png&originHeight=896&originWidth=1552&size=57670&status=done&style=stroke&width=776)

## Nacos 客户端解析

### 获取配置

获取配置的主要方法是 `NacosConfigService` 类的 `getConfigInner` 方法，通常情况下该方法直接从本地文件中取得配置的值，如果本地文件不存在或者内容为空，则再通过 `HTTP GET` 方法从远端拉取配置，并保存到本地快照中。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579162126460-7b1e8d9b-38da-4757-8585-c7aa95ab668a.png#align=left&display=inline&height=273&name=image.png&originHeight=546&originWidth=1292&size=55356&status=done&style=stroke&width=646)

当通过 `HTTP` 获取远端配置时，`Nacos` 提供了两种熔断策略，一是超时时间，二是最大重试次数，默认重试三次。

### 注册监听器

配置中心客户端对某个配置项注册监听器是很常见的需求，达到在配置项变更的时候执行回调的功能。

```java
iconfig.addListener(dataId, group, ml);
iconfig.getConfigAndSignListener(dataId, group, 1000, ml);
```

`Nacos` 可以通过以上方式注册监听器，它们内部的实现均是调用 `ClientWorker` 类的 `addCacheDataIfAbsent`。其中 `CacheData` 是一个维护配置项和其下注册的所有监听器的实例，私以为这个名字取得并不好，不容易理解。

所有的 `CacheData` 都保存在 `ClientWorker` 类中的原子 `cacheMap` 中，其内部的核心成员有：

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579166301180-ab180ddd-824d-4f94-8caf-0914f3ffeb19.png#align=left&display=inline&height=312&name=image.png&originHeight=624&originWidth=1202&size=44483&status=done&style=stroke&width=601)

其中，`content` 是配置内容，`MD5` 值是用来检测配置是否发生变更的关键，内部还维护着一个若干监听器组成的数组，一旦发生变更则依次回调这些监听器。

### 配置长轮询

`ClientWorker` 通过其下的两个线程池完成配置长轮询的工作，一个是单线程的 `executor`，每隔 `10ms` 按照每 `3000` 个配置项为一批次捞取待轮询的 `cacheData` 实例，将其包装成为一个 `LongPollingTask` 提交进入第二个线程池 `executorService` 处理。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579164883833-1910e7b9-9c31-43d5-8a2a-9cf7f796b8ad.png#align=left&display=inline&height=400&name=image.png&originHeight=840&originWidth=1566&size=129370&status=done&style=stroke&width=746)

该长轮询任务内部主要分为四步：

1. 检查本地配置，忽略本地快照不存在的配置项，检查是否存在需要回调监听器的配置项
1. 如果本地没有配置项的，从服务端拿，返回配置内容发生变更的键值列表
1. 每个键值再到服务端获取最新配置，更新本地快照，补全之前缺失的配置
1. 检查 `MD5` 标签是否一致，不一致需要回调监听器

如果该轮询任务抛出异常，等待一段时间再开始下一次调用，减轻服务端压力。另外，`Nacos` 在 `HTTP` 工具类中也有限流器的代码，通过多种手段降低轮询或者大流量情况下的风险。下文还会讲到，如果在服务端没有发现变更的键值，那么服务端会夯住这个 `HTTP` 请求一段时间（客户端侧默认传递的超时是 `30s`），以此进一步减轻客户端的轮询频率和服务端的压力。

## Nacos 服务端解析

### 配置 Dump

服务端启动时就会依赖 `DumpService` 的 `init` 方法，从数据库中 `load` 配置存储在本地磁盘上，并将一些重要的元信息例如 `MD5` 值缓存在内存中。服务端会根据心跳文件中保存的最后一次心跳时间，来判断到底是从数据库 `dump` 全量配置数据还是部分增量配置数据（如果机器上次心跳间隔是 `6h` 以内的话）。

全量 `dump` 当然先清空磁盘缓存，然后根据主键 `ID` 每次捞取一千条配置刷进磁盘和内存。增量 `dump` 就是捞取最近六小时的新增配置（包括更新的和删除的），先按照这批数据刷新一遍内存和文件，再根据内存里所有的数据全量去比对一遍数据库，如果有改变的再同步一次，相比于全量 `dump` 的话会减少一定的数据库 `IO` 和磁盘 `IO` 次数。

### 配置注册

`Nacos` 服务端是一个 `SpringBoot` 实现的服务，注册配置主要代码位于 `ConfigController` 和 `ConfigServletInner` 中。服务端一般是多节点部署的集群，因此请求一开始只会打到一台机器，这台机器将配置插入 `MySQL` 中进行持久化，这部分代码很简单不再赘述。

因为服务端并不是针对每次配置查询都去访问 `MySQL` 的，而是会依赖 `dump` 功能在本地文件中将配置缓存起来。因此当单台机器保存完毕配置之后，需要通知其他机器刷新内存和本地磁盘中的文件内容，因此它会发布一个名为 `ConfigDataChangeEvent` 的事件，这个事件会通过 `HTTP` 调用通知所有集群节点（包括自身），触发本地文件和内存的刷新。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579248126579-046cced9-653a-4498-81d6-700c6f7fbad1.png#align=left&display=inline&height=430&name=image.png&originHeight=860&originWidth=1244&size=157898&status=done&style=stroke&width=622)

### 处理长轮询

上文提到，客户端会有一个长轮询任务，拉取服务端的配置变更，那么服务端是如何处理这个长轮询任务的呢？源码逻辑位于 `LongPollingService` 类，其中有一个 `Runnable` 任务名为 `ClientLongPolling`，服务端会将受到的轮询请求包装成一个 `ClientLongPolling` 任务，该任务持有一个 `AsyncContext` 响应对象（`Servlet 3.0` 的新机制），通过定时线程池延后 `29.5s` 执行。

> 为什么比客户端 `30s` 的超时时间提前 `500ms` 返回是为了最大程度上保证客户端不会因为网络延时造成超时


![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579242276244-0d52478f-e87e-4a69-9dc8-97bc71592bb3.png#align=left&display=inline&height=364&name=image.png&originHeight=750&originWidth=1536&size=121607&status=done&style=stroke&width=746)

这里需要注意的是，在 `ClientLongPolling` 任务被提交进入线程池待执行的同时，服务端也通过一个队列 `allSubs` 保存了所有正在被夯住的轮询请求，这是因为在配置项被夯住的期间内，如果用户通过管理平台操作了配置项变更、或者服务端该节点收到了来自其他节点的 `dump` 刷新通知，那么都应立即取消夯住的任务，及时通知客户端数据发生了变更。

为了达到这个目的，`LongPollingService` 类继承自 `Event` 接口，实际上本身是个事件触发器，需要实现 `onEvent` 方法，其事件类型是 `LocalDataChangeEvent`。

当服务端在请求被夯住的期间接收到某项配置变更时，就会发布一个 `LocalDataChangeEvent` 类型的事件通知（注意同上文中的 `ConfigDataChangeEvent` 区别），之后会将这个变更包装成一个 `DataChangeTask` 异步执行，内容就是从 `allSubs` 中找出夯住的 `ClientLongPolling` 请求，写入变更强制其立即返回。

因此完整的流程如下，如果非接收请求的节点，那么忽略第一步持久化配置后开始：

![image.png](https://cdn.nlark.com/yuque/0/2020/png/698983/1579249384718-1ca773fc-2513-4b76-ad50-5a39ff3c48ca.png#align=left&display=inline&height=128&name=image.png&originHeight=288&originWidth=1672&size=47766&status=done&style=stroke&width=746)

## 全文总结

本文聚焦于 `Nacos` 作为配置中心的源码实现，包含了客户端和服务端两部分，内容基本覆盖了配置中心功能的关键点，既作为学习总结，也希望对阅读的朋友有所帮助。