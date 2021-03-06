---
layout: post
title: "回顾《深入理解 Java 虚拟机》 之内存模型和 volatile 关键字"
subtitle: "第五篇"
date: 2019-02-12
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - JVM
---

定义 `Java` 内存模型并不是一件容易的事情，这个模型必须定义得足够严谨，才能让 `Java` 的并发操作**不会产生歧义**；但是，也必须得足够宽松，使得虚拟机的实现能有足够的自由空间去利用硬件的各种特性（`寄存器`、`高速缓存`等）来获取更好的执行速度。经过长时间的验证和修补，在`JDK1.5`发布后，Java内存模型就已经成熟和完善起来了。

### 主内存和工作内存

`Java` 内存模型中规定了**所有的变量都存储在主内存中**，每条线程还有自己的`工作内存`（可以与前面将的处理器的高速缓存类比），线程的工作内存中**保存了该线程使用到的变量到主内存副本拷贝**，线程对变量的所有操作（`读取`、`赋值`）都必须在`工作内存`中进行，而**不能直接读写主内存中的变量**。不同线程之间无法直接访问对方工作内存中的变量，线程间变量值的传递均需要在主内存来完成，线程、主内存和工作内存的交互关系如下图所示

![](https://s2.ax1x.com/2019/02/12/kwTpOf.png)

### volatile 关键字

关键字 `volatile` 可以说是 `Java` 虚拟机提供的**最轻量级的同步机制**，我们来理解一下 `volatile` 的语义。

当一个变量定义为 `volatile` 之后，它将具备**两种特性**，

 1. 保证此变量对于所有线程的可见性，这里`可见性`是指当一条线程修改了这个变量的值，新值对于其他线程来说是立即得知的，普通变量不能做到这点，因为普通变量的值在线程之间传递均需要通过主内存来完成。
 2. 禁止指令重排序优化，这是通过插入内存屏障来实现的。禁止指令重排序优化，这是通过插入`内存屏障`来实现的。

#### 实现原理和机制

下面这段话摘自 **《深入理解Java虚拟机》**：

> *观察加入 volatile 关键字和没有加入volatile关键字时所生成的汇编代码发现，加入 volatile 关键字时，会多出一个lock前缀指令*

lock 前缀指令实际上相当于一个`内存屏障`（也称`内存栅栏`），内存屏障会提供三个功能：

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. 它会强制将对缓存的修改操作立即写入主存；
3. 如果是写操作，它会导致其他 `CPU` 中对应的缓存行无效。

#### 再谈可见性

可见性的实现是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是 `volatile` 变量都是如此，区别是 `volatile` 变量的特殊规则保证了新值能立即同步到主内存以及每次使用立即从主内存刷新，普通变量则不行。

`Java` 还有**两个关键词**可以实现**可见性**，分别是 `synchronized` 和 `final`。

其中 `synchronized` 的可见性是由 **“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中”** 这条规则获得的；

而 `final` 是指 **“被final修饰的字段在构造器中一旦初始化完成，并且构造器没有把this的引用传递出去，那么其他线程就能看见final字段的值”**

这里有个需要特别注意的地方，就是 `volatile` 变量的可见性并不保证 `volatile` 变量的运算是线程安全的，因为 `Java` 中运算并不是原子操作（例如被 `volatile` 关键字修饰的变量进行 ++ 操作）。

#### 再谈有序性

`volatile` 的有序性由**禁止指令重排序完成**，`synchronized` 的有序性由**每次只允许一条线程对其进行 `lock` 操作获得**。

但是如果所有有序性都要通过 `volatile` 或者 `synchronized` 来实现，那么会很繁琐，`Java` 中有一种 **“先行发生”（happen before）** 原则，依靠这个原则我们可以一揽子解决并发环境下两个操作之间是否可能存在冲突的问题。

**先行发生 （happen before）** 是 `Java` 内存模型中定义的两项操作之尖的**偏序关系**，发生 `操作 B` 之前，`操作A` 产生的影响能被`操作B`观察到。`Java` 内存模型下有一些天然的**先行发生（happen before）**关系：

 1. 程序次序规则：代码的控制流顺序
 2. `volatile` 变量原则
 3. 线程启动规则：`start` 方法先于此线程的每一个动作
 4. 线程终止操作：线程所有操作都先行与此线程的终止操作

#### volatile 关键字的性能

多数情况下，`volatile` 的总开销仍然比锁要低。和读取普通变量相比，读操作几乎一样，写操作因为多了插入内存屏障的步骤因此会慢一些。

### 参考资料

- 《深入理解Java虚拟机》 周志明著

### 许可协议

- 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>