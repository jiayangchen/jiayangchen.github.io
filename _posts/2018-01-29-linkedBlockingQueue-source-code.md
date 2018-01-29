---
layout: post
title: "LinkedBlockingQueue 源码分析"
subtitle: "JDK 1.8 中 LinkedBlockingQueue 的源码分析"
date: 2018-01-26
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
    - Java Concurrent
---

### 简介

LinkedBlockingQueue 是由链表实现的阻塞队列，按照源码注释中的说法既可以是 “无界的”（如果一开始没有指定容量大小，则为 Integer.MAX_VALUE），也可以指定大小，元素按照 FIFO 的形式来访问，队列头部为待的时间最久的元素，尾部则是最少，新元素插在尾部。大多数情况下，链表实现的阻塞队列比数组实现的队列具有更高的吞吐量，这是因为像 ArrayBlockingQueue 这样底层是由数组实现的阻塞队列在取值和插值的时候，会锁住整个 array ，而 LinkedBlockingQueue 在实现时对于取和插这两个不同的操作采用了不同的锁进行，但是在多线程环境下也有可能产生各种不可预料的执行后果。

### UML 类图
从类图中我们跟之前分析过的 PriorityQueue 作对比可以发现，阻塞队列跟普通队列相比，主要新增了 BlockingQueue 这个接口，我们下面就来看看这个接口中的方法。

![](http://o9oomuync.bkt.clouddn.com/LinkedBlockingQueue.png)

### 接口 BlockingQueue
从源码注释中可知，相比于普通的队列操作，阻塞队列最大的不同在于，当我们从 queue 中获取元素时 queue 为空时，线程会持续等待直到队列不为空为止，相当于一直阻塞在那儿，反之插入元素也是一样，如果 queue 是满的会一直阻塞到 queue 空缺出一个位置为止。阻塞队列不接受 null 值，如果 add、put、offer 方法接受 null 值会抛空指针异常。阻塞队列常用于生产者 - 消费者场景。阻塞队列的实现是线程安全的，但是对于批量操作则不一定保证，它也没有提供任何类似 close、shutdown 之类的操作来表示没有需要添加的元素了，

### 接口 BlockingQueue 方法
```java
//才队列头部取数据，为空的话等待指定时间
E poll(long timeout, TimeUnit unit)
        throws InterruptedException;
//插入数据，等待指定时间
boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
```

### LinkedBlockingQueue 源码分析

#### 一些属性
```java
    /* 队列元素数量，跟 ArrayBlockingQueue count 为 int 类型，因为 count 操作都是一把锁加锁进行，但是 LinkedBlockingQueue 中是两把锁，入队出队都会涉及对 count 的修改，因此这里使用了一个原子操作类
    来解决对同一个变量进行并发修改的线程安全问题 */
    private final AtomicInteger count = new AtomicInteger();

    /** 出队锁 */
    private final ReentrantLock takeLock = new ReentrantLock(); 

    /** 出队条件 */
    private final Condition notEmpty = takeLock.newCondition();

    /** 入队锁 */
    private final ReentrantLock putLock = new ReentrantLock();

    /** 入队等待条件 */
    private final Condition notFull = putLock.newCondition();
```

#### 构造器
```java
    // “无界”
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    //指定初始容量
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    //拿一个集合初始化
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

#### 核心方法 put
```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        // 有注释可知，这是赋值给局部变量的过程
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        // 执行可中断的锁获取操作，避免死锁。
        putLock.lockInterruptibly();
        try {
             // 队列满员时等待
            while (count.get() == capacity) {
                notFull.await();
            }
            // 否则插入队尾
            enqueue(node);
            // 更新 count 大小，返回的是旧值注意~
            c = count.getAndIncrement();
            //这里指的是队列中必须至少有一个空位时，才会通知 notFull
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        // 当c = 0时，即意味着之前的队列是空队列,现在新添加了一个新的元素之后队列不再为空,因此它会唤醒正在等待获取元素的线程。
        if (c == 0)
            signalNotEmpty();
    }

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //队列不为空了，可以获取元素
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

```

#### 核心方法 timed offer
该方法是限时等待插入操作，即在等待一定的时间内，如果队列有空间可以插入，那么就将元素入队列，然后返回 true,如果在过完指定的时间后依旧没有空间可以插入，那么就返回 false。

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        //将指定的时间长度转换为毫秒来进行处理
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                //如果设置的等待时间小于等于0，那么直接返回
                if (nanos <= 0)
                    return false;
                //否则等待预设时间
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            //返回旧值注意！！
            c = count.getAndIncrement(); 
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

#### 核心方法 take
```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            // 为空则等待
            while (count.get() == 0) {
                notEmpty.await();
            }
            // 出队元素
            x = dequeue();
            // 队列元素个数完成原子操作-1, count 会因为插入元素的线程和获取元素的线程而发生并发修改操作
            //注意！这里 c 返回的是旧值
            c = count.getAndDecrement();
            // 如果还有元素那么队列依旧不为空
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //如果旧值是空，现在已经增加了一个元素通知获取元素的线程可以来取元素了
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

### LinkedBlockingQueue 和 ArrayBlockingQueue 的对比
ArrayBlockingQueue 底层基于数组而实现，并且在创建时需要指定容量大小，在完成后就会立即在内存分配固定大小容量的数组空间，所以它是是一个有界的阻塞队列；

而LinkedBlockingQueue可以由用户指定最大存储容量，也可以无需指定，如果不指定则最大存储容量将是Integer.MAX_VALUE，即可以看作是一个无界的阻塞队列，由于其节点的创建都是动态创建，在节点出队之后可以被 GC 回收。

ArrayBlockingQueue 因其有界性，能够更好的对于性能进行预测，而 LinkedBlockingQueue 由于没有限制大小，当任务非常多的时候，不停地向队列中存储，就有可能导致内存溢出的情况发生。

其次，ArrayBlockingQueue 在入队和出队操作中，使用的是同一把锁，所以即使在多核 CPU 的情况下，其读写操作都无法做到并行，而 LinkedBlockingQueue 的读写操作使用两把锁，一把出队锁，一把入队锁，它们之间的操作互相不受干扰，因此两种操作可以并行完成，故 LinkedBlockingQueue 的吞吐量要高于 ArrayBlockingQueue。

### 在线程池中的使用场景
```java
    //下面的代码是 Executors 创建 FixedThreadPool 的代码，其使用了 LinkedBlockingQueue 来作为任务队列。
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

之所以使用 LinkedBlockingQueue 作为任务阻塞队列的原因就在于它的无界性。因为线程大小固定的线程池，其线程的数量是不具备伸缩性的，当任务非常繁忙、所有的线程都处于工作状态，这时候如果使用一个有界阻塞队列来进行处理，可能很快就会导致队列满员，从而触发任务拒绝策略抛出 RejectedExecutionException，而使用无界队列由于其容量可以伸缩，可以很好地适应任务繁忙的情况，即使任务非常多，也可以进行动态扩容，当任务被处理完成之后，队列中的节点也会被随之被 GC 回收，非常灵活。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>
