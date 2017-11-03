---
layout: post
title: "深入理解java虚拟机 —— Java GC算法"
subtitle: "进一步了解GC中的各种算法"
date: 2017-04-27
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 读书笔记
    - 深入理解Java虚拟机
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@emilep?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Émile Perron"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Émile Perron</span></a>

### 标记清除
第一步标记所有需要回收的对象，然后进行回收。不足一是效率问题，标记和清除两个过程的效率都不高；二是标记清除之后会产生大量不连续碎片，后续分配大对象时会无法找到足够的连续内存而不得不触发GC

### 复制算法
为解决效率问题，将内存容量划分为大小相等的两块，每次只使用其中的一块，当这块用完了，将其中存活的对象复制到另外一块，使得每次都只对半块进行回收，不会担心碎片问题。但代价是可用的内存变为原先的一半。商业虚拟机采用这种方法回收新生代，但并不是按照1:1来划分两快内存，Hotspot按照8:1划分。

### 标记整理
复制算法在对象存活率较高时就需要进行较多的复制，效率会很低。根据老年代的特点，提出了标记整理算法，先进性标记清除，然后让所有存活的对象都向一端移动，然后清理掉端边界以外的内存。

### 分代收集
#### Java 堆内存
我们有必要了解堆内存在JVM内存模型的角色。在运行时，Java的实例被存放在堆内存区域。当一个对象不再被引用时，满足条件就会从堆内存移除。在垃圾回收进程中，这些对象将会从堆内存移除并且内存空间被回收。堆内存以下三个主要区域：

新生代（Young Generation）
Eden空间（Eden space，任何实例都通过Eden空间进入运行时内存区域）
S0 Survivor空间（S0 Survivor space，存在时间长的实例将会从Eden空间移动到S0 Survivor空间）
S1 Survivor空间 （存在时间更长的实例将会从S0 Survivor空间移动到S1 Survivor空间）
老年代（Old Generation）实例将从S1提升到Tenured（终身代）
永久代（Permanent Generation）包含类、方法等细节的元信息

根据对象的存活周期不同将内存划分成不同的快，一般有新生代和老年代，新生代使用复制，老年代使用标记整理

![image](http://images.cnitblog.com/blog/587773/201409/061921034534396.png)

从图中可以看出： 堆大小 = 新生代 + 老年代。其中，堆的大小可以通过参数 –Xms、-Xmx 来指定。
（本人使用的是 JDK1.6，以下涉及的 JVM 默认值均以该版本为准。）默认的，新生代 ( Young ) 与老年代 ( Old ) 的比例的值为 1:2 ( 该值可以通过参数 –XX:NewRatio 来指定 )，即：新生代 ( Young ) = 1/3 的堆空间大小。老年代 ( Old ) = 2/3 的堆空间大小。其中，新生代 ( Young ) 被细分为 Eden 和 两个 Survivor 区域，这两个 Survivor 区域分别被命名为 from 和 to，以示区分。

默认的，Edem : from : to = 8 : 1 : 1 ( 可以通过参数 –XX:SurvivorRatio 来设定 )，即： Eden = 8/10 的新生代空间大小，from = to = 1/10 的新生代空间大小。JVM 每次只会使用 Eden 和其中的一块 Survivor 区域来为对象服务，所以无论什么时候，总是有一块 Survivor 区域是空闲着的。因此，新生代实际可用的内存空间为 9/10 ( 即90% )的新生代空间。

当一个对象被判定为 "死亡" 的时候，GC 就有责任来回收掉这部分对象的内存空间。新生代是 GC 收集垃圾的频繁区域。当对象在 Eden ( 包括一个 Survivor 区域，这里假设是 from 区域 ) 出生后，在经过一次 Minor GC 后，如果对象还存活，并且能够被另外一块 Survivor 区域所容纳( 上面已经假设为 from 区域，这里应为 to 区域，即 to 区域有足够的内存空间来存储 Eden 和 from 区域中存活的对象 )，则使用复制算法将这些仍然还存活的对象复制到另外一块 Survivor 区域 ( 即 to 区域 ) 中，然后清理所使用过的 Eden 以及 Survivor 区域 ( 即 from 区域 )，并且将这些对象的年龄设置为1，以后对象在 Survivor 区每熬过一次 Minor GC，就将对象的年龄 + 1，当对象的年龄达到某个值时 ( 默认是 15 岁，可以通过参数 -XX:MaxTenuringThreshold 来设定 )，这些对象就会成为老年代。<b>但这也不是一定的，对于一些较大的对象 ( 即需要分配一块较大的连续内存空间 ) 则是直接进入到老年代。</b>

<b>需要注意的是HotSpot虚拟机使用了两种技术来加快内存分配。他们分别是是”bump-the-pointer“和“TLABs（Thread-Local Allocation Buffers）”。</b>

Bump-the-pointer技术跟踪在伊甸园空间创建的最后一个对象。这个对象会被放在伊甸园空间的顶部。如果之后再需要创建对象，只需要检查伊甸园空间是否有足够的剩余空间。如果有足够的空间，对象就会被创建在伊甸园空间，并且被放置在顶部。这样以来，每次创建新的对象时，只需要检查最后被创建的对象。这将极大地加快内存分配速度。但是，如果我们在多线程的情况下，事情将截然不同。如果想要以线程安全的方式以多线程在伊甸园空间存储对象，不可避免的需要加锁，而这将极大地的影响性能。TLABs 是HotSpot虚拟机针对这一问题的解决方案。该方案为每一个线程在伊甸园空间分配一块独享的空间，这样每个线程只访问他们自己的TLAB空间，再与bump-the-pointer技术结合可以在不加锁的情况下分配内存。

#### 如果老年代的对象需要引用一个新生代的对象，会发生什么呢？

为了解决这个问题，老年代中存在一个”card table“，他是一个512 byte大小的块。所有老年代的对象指向新生代对象的引用都会被记录在这个表中。当针对新生代执行GC的时候，只需要查询card table来决定是否可以被收集，而不用查询整个老年代。这个card table由一个write barrier来管理。write barrier给GC带来了很大的性能提升，虽然由此可能带来一些开销，但GC的整体时间被显著的减少。

### 参考资料
* 《深入理解Java虚拟机》 周志明著