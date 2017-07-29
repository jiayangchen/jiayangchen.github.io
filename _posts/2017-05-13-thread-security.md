---
layout: post
title: "深入理解java虚拟机 —— Java 线程安全"
subtitle: ""
date: 2017-05-13
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 深入理解Java虚拟机
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@emilep?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Émile Perron"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Émile Perron</span></a>

### 线程安全的实现方法
#### 互斥同步
同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一个线程使用，而互斥是实现同步的一种手段，临界区、互斥量、信号量都是主要的互斥实现手段。

#### 第一种方法：Synchronized 关键字
在Java中，最基本的互斥同步手段就是 Synchronized 关键字，Synchronized 关键字如果是修饰代码块的话，经过编译之后会在同步块的前后分别生成 monitorenter 和 monitorexit 这两个字节码指令，这两个字节码都需要一个reference类型的参数来指明要锁定和解锁的对象。如果Java程序中的 Synchronized 明确指定了对象参数，那就是这个对象的reference，如果没有指定，就根据 Synchronized 修饰的实例方法还是类方法，去取对应的对象实例或者Class对象来作为锁对象。如果是修饰方法的话，方法的同步并没有通过指令monitorenter和monitorexit来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

在执行monitorenter指令时，首先尝试获取对象的锁，如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，就把锁的计数器加一；相应的执行monitorexit时将计数器减一，当计数器为0时，锁就被释放。如果获取对象锁失败，那么当前线程就需要阻塞等待，直到对象锁被另一个线程释放为止。

注：Synchronized 同步快对于同一条线程来说是可重入的，不会出现自己把自己锁死的问题。其次，同步快在进入的线程执行完成之前，会阻塞后面其他线程的进入，因为java线程是映射到OS原生线程上的，阻塞和唤醒都需要切换到内核态，消耗很多的处理器时间，所以Synchronized是一个重量级的操作，应尽量避免，虚拟机自身对这种做了优化，例如在通知OS阻塞线程前加入一段自旋等待的过程，避免频繁切入到内核态中。

#### 第二种方法：重入锁 ReentrantLock
基本用法和Synchronized相似，一个表现为API层面的互斥锁，使用lock() 和 unlock()方法配合try/finally 语句块来完成；另一个表现为原生语法层面的互斥锁，不过相比于Synchronized，ReentrantLock增加了一些高级功能，主要有三项：
1. 等待可中断：当持有锁的线程长时间不释放锁的时候，等待的线程可以放弃等待转而处理其他事情
2. 可实现公平锁：多个线程在等待锁时，必须按照申请锁的顺序来一次获得锁。Synchronized的锁是非公平的，ReentrantLock的锁默认情况下也是非公平的。
3. 锁可绑定多个条件：ReentrantLock对象可以同时绑定多个Condition对象，而在Synchronized中，多个条件关联需要额外添加锁，ReentrantLock无需这样做，只需要多次调用new Condition()即可

#### 性能对比
多线程环境下，Synchronized的吞吐量下降严重，而ReentrantLock基本保持在同一个比较稳定的水平上，JDK1.6之后，Synchronized和ReentrantLock性能基本上完成持平了，建议使用Synchronized来进行同步。

#### 非阻塞同步
互斥同步最主要的问题是在于进行线程阻塞和唤醒所带来的性能问题，因此这种同步也被称为阻塞同步。从处理方式来说，互斥同步属于一种悲观的并发策略。现在我们还有一种基于冲突检测的乐观并发策略，就是先进行操作，如果没有其他线程争用共享数据，那操作就成了；如果有共享数据争用就产生了冲突，再采取其他补偿措施（最常见的就是不断重试直到成功为止），这种乐观的并发策略的许多实现都不需要把线程挂起，因此这种同步操作被称为非阻塞同步。

我们需要操作和冲突检测具备原子性，所以需要硬件的帮助。JDK1.5之后，java程序中可以使用CAS操作。CAS有三个参数，分别是内存地址V，旧预期值A与新值B，当且仅当V符合旧预期值A时，用B更新V。

#### 无同步方案
所有的可重入代码都是线程安全的，但不是所有线程安全的代码都是可重入的。可重入方法不依赖于存储在堆上的数据和公有的系统资源。如果一个方法它的返回结果是可预测的，只要输入了相同的参数就能返回相同的结果，那么它就满足可重入的要求，当然就是线程安全的。

另一个方法就是线程本地储存，把共享数据的可见范围限制在同一个线程内，大部分使用消息队列的架构模式，如生产者消费者都会将产品的消费过程尽量在一个线程中消费完。

### 参考资料
* 《深入理解Java虚拟机》 周志明著