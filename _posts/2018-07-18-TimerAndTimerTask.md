---
layout: post
title: "Timer & TimerTask 源码分析"
subtitle: "介绍实现定时任务的 Timer 和 TimerTask 类的实现，为什么还要写个源码分析呢？主要是由于这部分的实现非常有借鉴意义"
date: 2018-07-18
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
---

承接上一篇，看一看 Timer 和 `TimerTask` 内部的实现。之前说了我自己极少使用这个，目前在 Java 开发中使用 Timer 应该也非常少见了，既然是这样一个夕阳组件，为什么还要写个源码分析呢？主要是由于这部分的实现非常有借鉴意义，如果你工作中需要开发一个自动化流程，让它每一步都能定时执行，那么其实现方式和 Timer、TimerTask 的内部实现其实大同小异，都是维护一个 TaskQueue，然后依次获取任务，其中涉及到的一些并发处理技巧，很适合初学者学习。

OK，首先还是老规矩先看注释，了解作者对于这个类的介绍。在 Timer 类中，作者指出了 **每一个 Timer 对象对应一个线程**，因为单线程执行任务的原因，所以如果前一个任务的执行时间过长，会阻塞后一个任务。Timer 类在所有任务执行完毕之后可以被回收，但是并不规定回收时间，也许会在很久之后。另外，如果运行过程中抛异常，那么后续任务直接 crash 不会继续执行。Timer 是线程安全的，多个线程共享同一个 Timer 对象，使用者不需要另做同步。最后，Timer 执行任务不做实时保证，因为它是通过 wait() 等待。综上，作者在注释最后也推荐了 JDK 1.5 之后加入的 ScheduledThreadPoolExecutor 作为 Timer 的替代品。

## TaskQueue
首先要介绍一下 Timer 内部的这个 TaskQueue，它是作为构造器的参数传递给 TimerThread 对象的。
```java
private final TaskQueue queue = new TaskQueue();
private final TimerThread thread = new TimerThread(queue);
```
先看一看 TaskQueue 类长什么样，内部很简单
```java
class TaskQueue {
    //原来里面有一个 TimerTask 的数组，注释中指出这个队列是一个优先级队列（平衡二叉堆），按照执行时间进行判断，queue[0]是执行时间最早的任务，反之最晚
    private TimerTask[] queue = new TimerTask[128];
    //加入任务的方法，如果queue满容就扩大两倍
    void add(TimerTask task) {
        // Grow backing store if necessary
        if (size + 1 == queue.length)
            queue = Arrays.copyOf(queue, 2*queue.length);

        queue[++size] = task;
        //根据二叉堆特性调整
        fixUp(size);
    }
    //还记得二叉堆是怎么删堆顶元素的么，堆顶放到堆末尾，size--，然后fix堆结构
    void removeMin() {
        queue[1] = queue[size];
        queue[size--] = null;  // Drop extra reference to prevent memory leak
        fixDown(1);
    }
    //调整堆结构的方法
    private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
                j++; // j indexes smallest kid
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
}
```

## TimerThread
这是 Timer 执行任务的线程，继承自 Thread，我们看看内部的实现
```java
class TimerThread extends Thread {
    //这个标记的含义是还有没有指向 Timer 对象的活跃引用，如果这个标记为 false 并且任务队列里没有任务了，就说明我们可以停止 Timer 的运行了
    boolean newTasksMayBeScheduled = true;
    //run方法，里面调用了 mainLoop，看名字猜测应该就是循环取任务执行
    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }
    //说实话第一次看到这部分源码很亲切，因为工作中接触到的一个需求的代码就是借鉴的这个实现，甚至还要更复杂一些
    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    // Wait for queue to become non-empty
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait(); //等待任务进入
                    if (queue.isEmpty())
                        break; // Queue is empty and will forever remain; die

                    // Queue nonempty; look at first evt and do the right thing
                    long currentTime, executionTime;
                    task = queue.getMin(); //取队首的任务
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue;  // No action required, poll queue again
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        if (taskFired = (executionTime<=currentTime)) { //是不是到了执行时间
                            if (task.period == 0) { // Non-repeating, remove
                                queue.removeMin();
                                task.state = TimerTask.EXECUTED;
                            } else { // Repeating task, reschedule
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period); //重复执行
                            }
                        }
                    }
                    if (!taskFired) // Task hasn't yet fired; wait
                        queue.wait(executionTime - currentTime); //还没到执行时间等待时间差值
                }
                if (taskFired)  // Task fired; run it, holding no locks
                    task.run(); //执行这个任务
            } catch(InterruptedException e) {
            }
        }
    }
}
```

## Timer
上述两个主要的实现方法讲完了，其实 Timer 就没剩下什么东西了。先看一下源码，下面这个小细节可以注意，在你自己使用线程池创建线程的时候，一个好习惯也是要这样给每个线程命名。
```java
private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
private static int serialNumber() {
    return nextSerialNumber.getAndIncrement();
}
```
### Timer.constructor
接下去就是 Timer 的几个不同的构造器了，没什么好说的，看看就行了。
```java
public Timer() {
        this("Timer-" + serialNumber());
    }
public Timer(boolean isDaemon) {
        this("Timer-" + serialNumber(), isDaemon);
    }
public Timer(String name) {
        thread.setName(name);
        thread.start();
    }
public Timer(String name, boolean isDaemon) {
        thread.setName(name);
        thread.setDaemon(isDaemon);
        thread.start();
    }
```
### Timer.schedule
接下去是 Timer 的 schedule 方法，有几个不同的重载形式。
```java
//注意，这里 delay 是采用系统时间加上delay的毫秒数来进行的，如果外部人员更改系统时间就会导致编写的 Timer 程序无法正常运行
public void schedule(TimerTask task, long delay) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        sched(task, System.currentTimeMillis()+delay, 0);
    }
public void schedule(TimerTask task, Date time) {
    sched(task, time.getTime(), 0);
}
public void schedule(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, -period);
    }
public void schedule(TimerTask task, Date firstTime, long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), -period);
    }
//以固定速率执行，在delay之后开始
public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, period);
    }
//这里则是指定了开始的时间
public void scheduleAtFixedRate(TimerTask task, Date firstTime,
                                    long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), period);
    }
```

### Timer.sched
上述看到的 schedule 方法，其最后都是调用 sched 方法来实现功能，下面我们就看看这个方法。
```java
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");

    // Constrain value of period sufficiently to prevent numeric
    // overflow while still being effectively infinitely large.
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");

        synchronized(task.lock) {
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            task.nextExecutionTime = time;
            task.period = period;
            task.state = TimerTask.SCHEDULED;
        }
        //加入队列
        queue.add(task);
        if (queue.getMin() == task)
            queue.notify(); //唤醒，可以去执行了
    }
}
```

## TimerTask 
这个类就更简单了，本质是一个实现了 Runnable 接口的抽象类，使用的时候需要具体的实现子类继承它，重写其中的 run 方法，TimerTask 内部主要定义了几个任务状态标识 VIRGIN、CANCELED、SCHEDULED、EXECUTED 等，外加执行时间属性 nextExecutionTime、间隔 period，还有一个锁标识 lock。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>



