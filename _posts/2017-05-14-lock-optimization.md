---
layout: post
title: "深入理解java虚拟机 —— Java 锁优化"
subtitle: ""
date: 2017-05-14
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 读书笔记
    - 深入理解Java虚拟机
---

高效并发是JDK1.5到JDK1.6的一个重要改进，Hotspot虚拟机开发团队在这个版本上花了大量精力去实现各种锁优化技术，如自旋适应锁、锁消除、锁粗化、轻量级锁和偏向锁，这些技术都是为了在线程之间更高效的共享数据以及解决竞争的问题。

### 自旋锁与自适应自旋
许多应用上，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得，我们可以让后面请求锁的那个线程“稍等一会”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。

优点是如果锁被占用时间很短，自旋等待的效果就会非常好；反之如果锁被占用时间很长，自旋线程只会浪费处理器资源而不做任何有效的工作。因此自旋必须有一定的限度，如果超过限度仍然没有获得锁，就应当使用传统方式挂起线程，自旋次数默认是10。

JDK1.6中引入自旋锁并不意味着自旋的时间不在固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定的。如果同一个锁对象上，自旋等待刚刚成功获得过锁，那么这次有可能继续成功，进而它将允许自旋等待持续更长的时间；反之如果很少成功获得，那么这个锁可能就会省略掉自旋过程。

### 锁消除
对于一些要求同步，但是检测到不可能存在共享数据竞争的锁进行消除，主要依赖于逃逸分析的数据支持实现这个功能

### 锁粗化
如果一系列连续操作对同一个对象反复加锁解锁，例如加锁操作出现在循环体中，那么这会导致不必要的性能损耗。如果虚拟机探测到这种情况，将会把锁的范围扩大到整个操作序列的外部，这样只需要加一次锁就够了。

### 轻量级锁
使用CAS更新对象的Mark World

### 偏向锁

### 参考资料
* 《深入理解Java虚拟机》 周志明著

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@emilep?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Émile Perron"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Émile Perron</span></a>