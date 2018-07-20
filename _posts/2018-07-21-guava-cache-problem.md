---
layout: post
title: "GuavaCache RemovalListener 失灵"
subtitle: "日常问题记录系列"
date: 2018-07-20
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 问题排查
---

## 问题
最近接触到 Google 的 Java 工具包 Guava，确实很好用，特别是其中的 GuavaCache 算是经常使用到的本地缓存，这次遇到一个需求是希望在 xxx min 之后做一个延时操作，虽然可以开一个定时任务做，但是由于当时代码里正好使用到了 GuavaCache，想起来可以基于 Key 的过期做一个回调方法不就行了嘛，猜测 GuavaCache 肯定是支持这样的回调 API 的，一查果然如此。

于是乎，代码写得很快，如下：
```java
private static class MyRemovalListener implements RemovalListener<Integer, Integer> {  
    @Override  
    public void onRemoval(RemovalNotification<Integer, Integer> notification) {  
        doSomething();
    }  
}  

public static Cache<Integer, Integer> cache = CacheBuilder.newBuilder()
.expireAfterWrite(10, TimeUnit.MINUTES)
.removalListener(new MyRemovalListener())
.build();

```
看上去应该没什么问题，只是测试下来发现回调并不触发，如果手动调用 invalidate(key) 的话，那么就会触发回调，但是这不符合我的需求啊，我是希望 key 在 10 min 过期之后能自己触发回调，而且这 API 既然设计成这样，没道理不能触发，于是开始寻找问题

## 原因
在 `Stack Overflow` 上找到了原因，原来是 GuavaCache 并不保证在过期时间到了之后立刻删除该 Key，如果你此时去访问了这个 Key，它会检测是不是已经过期，过期就删除它，所以过期时间到了之后你去访问这个 Key 会显示这个 Key 已经被删除，但是如果你不做任何操作，那么在 10 min 到了之后也许这个 Key 还在内存中。GuavaCache 选择这样做的原因也很简单，如下：

> The reason for this is as follows: if we wanted to perform Cache maintenance continuously, we would need to create a thread, and its operations would be competing with user operations for shared locks. Additionally, some environments restrict the creation of threads, which would make CacheBuilder unusable in that environment.

这样做既可以保证对 Key 读写的正确性，也可以节省资源，减少竞争。