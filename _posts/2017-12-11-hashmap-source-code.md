---
layout: post
title: "HashMap 源码分析"
subtitle: ""
date: 2017-12-11
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
---

### UML 类图
![](http://o9oomuync.bkt.clouddn.com/hashmaphashmap%E7%B1%BB%E5%9B%BE.png)

不知道大家还记不记得在 ArrayList 那篇文章中，我谈到说不定存在 AbstractSet、AbstractMap 等抽象类的事情，那是基于对类层设计的猜想，现在看到 HashMap 的层级类图之后，我们会发现确实存在 AbstractMap 这个抽象类，也印证了整个 Java 集合类的设计确实遵循严格的规范，这是值得我们仔细体会和学习的。

### Map
老规矩，我们分析 HashMap 之前还是先来看看其他的接口和抽象类，先是最顶层的 Map 接口。

#### Introduction

> An object that maps keys to values.  A map cannot contain duplicate keys; each key can map to at most one value.

Map 是一种存储键值对的对象，其中键值 key 不允许重复，目的就是一个 key 最多只能对应一个值。

> This interface takes the place of the <tt>Dictionary</tt> class, which was a totally abstract class rather than an interface.

由此可见，Map 接口是取代了老式的 Dictionary 抽象类，由接口实现更显轻量、灵活，抽象类会有很多制约，例如不允许多重继承、强制子类实现抽象方法等。

> The <tt>Map</tt> interface provides three <i>collection views</i>, which allow a map's contents to be viewed as a set of keys, collection of values, or set of key-value mappings.  The <i>order</i> of a map is defined as the order in which the iterators on the map's collection views return their elements.  Some map implementations, like the <tt>TreeMap</tt> class, make specific guarantees as to their order; others, like the <tt>HashMap</tt> class, do not.

这段说明了 Map 提供了三种形式的集合视图：
1. keys 的集合
2. values 的集合
3. <key,value> 对的集合

并且，TreeMap 是保证插入顺序的，HashMap 则并不保证。

#### Methods
```java
int size(); //返回 map 键值对数目，如果超出了 Integer.MAX_VALUE 则返回 Integer.MAX_VALUE
boolean containsKey(Object key); // key==null ? k==null : key.equals(k) 最多支持一个 key 为 null
Set<K> keySet(); //值得注意的是，如果迭代过程中 map 被修改了（除却迭代器自身的修改行为），那么返回的结果是 undefined
... //其他方法不说了
```

### AbstractMap
#### Introduction

#### Methods

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>