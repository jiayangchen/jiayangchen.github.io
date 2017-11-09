---
layout: post
title: "深入理解java虚拟机 —— Java内存模型和volatile关键字"
subtitle: "定义Java内存模型并不是一件容易的事情，这个模型必须定义得足够严谨，才能让Java的并发操作不会产生歧义；但是，也必须得足够宽松，使得虚拟机的实现能有足够的自由空间去利用硬件的各种特性（寄存器、高速缓存等）来获取更好的执行速度。经过长时间的验证和修补，在JDK1.5发布后，Java内存模型就已经成熟和完善起来了。"
date: 2017-05-04
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 读书笔记
    - 深入理解Java虚拟机
    - 面试
---

定义Java内存模型并不是一件容易的事情，这个模型必须定义得足够严谨，才能让Java的并发操作不会产生歧义；但是，也必须得足够宽松，使得虚拟机的实现能有足够的自由空间去利用硬件的各种特性（寄存器、高速缓存等）来获取更好的执行速度。经过长时间的验证和修补，在JDK1.5发布后，Java内存模型就已经成熟和完善起来了。

### 主内存和工作内存
Java内存模型中规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存（可以与前面将的处理器的高速缓存类比），线程的工作内存中保存了该线程使用到的变量到主内存副本拷贝，线程对变量的所有操作（读取、赋值）都必须在工作内存中进行，而不能直接读写主内存中的变量。不同线程之间无法直接访问对方工作内存中的变量，线程间变量值的传递均需要在主内存来完成，线程、主内存和工作内存的交互关系如下图所示
![image](http://images.cnitblog.com/i/475287/201403/091134177063947.jpg)

### volatile变量
关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制，我们来理解一下volatile的语义。
当一个变量定义为volatile之后，它将具备两种特性，一是保证此变量对于所有线程的可见性，这里“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是立即得知的，普通变量不能做到这点，因为普通变量的值在线程之间传递均需要通过主内存来完成。

#### 实现原理和机制
下面这段话摘自《深入理解Java虚拟机》：
<b>“观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令”</b>

lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3个功能：
1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. 它会强制将对缓存的修改操作立即写入主存；
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

#### 再谈可见性
可见性的实现是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此，区别是volatile变量的特殊规则保证了新值能立即同步到主内存以及每次使用立即从主内存刷新，普通变量则不行。

Java还有两个关键词可以实现可见性，分别是synchronized和final，其中synchronized的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中”这条规则获得的；而final指“被final修饰的字段在构造器中一旦初始化完成，并且构造器没有把this的引用传递出去，那么其他线程就能看见final字段的值”

这里有个需要特别注意的地方，就是volatile变量的可见性并不保证volatile变量的运算是线程安全的，因为java中运算并不是原子操作。

第二个特性是禁止指令重排序优化，这是通过插入内存屏障来实现的。

#### 再谈有序性
volatile的有序性由禁止指令重排序完成，Synchronized的有序性由每次只允许一条线程对其进行lock操作获得。但是如果所有有序性都要通过volatile或者Synchronized来实现，那么会很繁琐，java中有一种“先行发生”原则，依靠这个原则我们可以一揽子解决并发环境下两个操作之间是否可能存在冲突的问题。

先行发生是java内存模型中定义的两项操作之尖的偏序关系，发生操作B之前，操作A产生的影响能被操作B观察到。java内存模型下有一些“天然的”先行发生关系：
* 程序次序规则：代码的控制流顺序
* volatile变量原则
* 线程启动规则：start方法先于此线程的每一个动作
* 线程终止操作：线程所有操作都先行与此线程的终止操作

#### volatile关键字的性能
多数情况下，volatile的总开销仍然比锁要低。和读取普通变量相比，读操作几乎一样，写操作因为多了插入内存屏障的步骤因此会慢一些。

### 参考资料
* 《深入理解Java虚拟机》 周志明著

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang[AT]foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@emilep?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Émile Perron"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Émile Perron</span></a>