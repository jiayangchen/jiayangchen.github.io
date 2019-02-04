---
layout: post
title: "ThreadPoolExecutor 核心源码解析"
subtitle: "本文介绍 ThreadPoolExecutor 源码的关键部分，execute、addWorker、runWorker、Worker"
date: 2019-02-04
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java Tech
    - Java Concurrent
    - JDK 源码分析
---

本文只介绍 `ThreadPoolExecutor` 源码的关键部分，开篇会先介绍  `ThreadPoolExecutor` 中的一些核心常量定义，然后选取线程池工作周期中的几个关键方法分析其源码实现。其实，看 `JDK` 源码的最好途径就是看类文件注释，作者把想说的全都写在里面了。

### 一些重要的常量

`ThreadPoolExecutor` 内部作者采用了一个 `32bit` 的 `int` 值来表示线程池的运行状态`（runState）`和当前线程池中的线程数目`（workerCount）`，这个变量取名叫 `ctl`（`control` 的缩写），其中高 `3bit` 表示允许状态，低 `29bit`表示线程数目（最多允许 `2^29 - 1` 个线程）。

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3; // 29 位
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 线程池最大容量

    // runState is stored in the high-order bits
	// 定义的线程池状态常量
	// 111+29个0，值为 -4 + 2 + 1 = -1（不懂的面壁）
    private static final int RUNNING    = -1 << COUNT_BITS; 
	// 000+29个0
    private static final int SHUTDOWN   =  0 << COUNT_BITS; 
	// 001+29个0
    private static final int STOP       =  1 << COUNT_BITS; 
	// 010+29个0
    private static final int TIDYING    =  2 << COUNT_BITS; 
	// 011+29个0
    private static final int TERMINATED =  3 << COUNT_BITS; 

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; } // 得到线程池状态
    private static int workerCountOf(int c)  { return c & CAPACITY; } // 得到线程池线程数
    private static int ctlOf(int rs, int wc) { return rs | wc; } // 反向构造 ctl 的值
```

因为代表线程池状态的常量可以通过值的大小来表示先后关系`（order）`，因此后续源码中会有：

```java
rs >= SHUTDOWN // 那就表示SHUTDOWN、 STOP or TIDYING or TERMINATED，反正不是 RUNNING
```

理解上述的常量意义有助于后面理解源码。

### 讨论线程池的状态转换

从第一节我们已经知道了线程池分为五个状态，下面我们聊聊这五个状态分别限制了线程池能执行怎样的行为：

1. `RUNNING：`可以接受新任务，且执行 `Queue` 中的任务
2. `SHUTDOWN：`不再接受新的任务，但能继续执行 `Queue` 中已有的任务
3. `STOP：`不再接受新的任务，且也不再执行 `Queue` 中已有的任务
4. `TIDYING：`所有任务完成，`workCount=0`，线程池状态转为 `TIDYING` 且会执行 `hook method`，即 `terminated()`
5. `TERMINATED：``hook method` `terminated()` 执行完毕之后进入的状态

### 线程池的关键逻辑

![](http://ww1.sinaimg.cn/large/c3beb895gy1fzuivjhq34j20pz0mn43z.jpg)

上图总结了 `ThreadPoolExecutor` 源码中的关键性步骤，正好对应我们此次解析的核心源码（上图出处见水印）。

1. `execute` 方法用来向线程池提交 `task`，这是用户使用线程池的第一步。如果线程池内未达到 `corePoolSize` 则新建一个线程，将该 `task` 设置为这个线程的 `firstTask`，然后加入 `workerSet` 等待调度，这步需要获取全局锁 `mainLock`
2. 已达到 `corePoolSize` 后，将 `task` 放入阻塞队列
3. 若阻塞队列放不下，则新建新的线程来处理，这一步也需要获取全局锁 `mainLock`
4. 当前线程池 `workerCount` 超出 `maxPoolSize` 后用 `rejectHandler` 来处理

我们可以看到，线程池的设计使得在 `2` 步骤时避免了使用全局锁，只需要塞进队列返回等待异步调度就可以，仅剩下 `1` 和 `3` 创建线程时需要获取全局锁，这有利于线程池的效率提升，因为一个线程池总是大部分时间在步骤 `2` 上，否则这线程池也没什么存在的意义。

### 源码分析

本文只分析 `execute`，`addWorker`，`runWorker`，三个核心方法和一个 `Worker` 类，看懂了这几个，其实其他的代码都能看懂。

#### Worker 类

```java
	// 继承自 AQS 实现简单的锁控制
 	private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        // worker 运行所在的线程
        final Thread thread;
        // 赋予该线程的第一个 task，可能是 null，如果不是 null 就运行这个，
		// 如果是 null 就通过 getTask 方法去 Queue 里取任务
        Runnable firstTask;
        // 线程完成的任务数量
        volatile long completedTasks;

        Worker(Runnable firstTask) {
		// 限制线程直到 runWorker 方法前都不允许被打断
            setState(-1); 
            this.firstTask = firstTask;
			// 线程工厂创建线程
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
			// 线程内部的 run 方法调用了 runWorker 方法
            runWorker(this);
        }
	}
```

#### execute 方法

```java
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
		// 如果当前线程数小于 corePoolSize
        if (workerCountOf(c) < corePoolSize) {
		// 调用 addWorker 方法新建线程，如果新建成功返回 true，那么 execute 方法结束
            if (addWorker(command, true))
                return;
			// 这里意味着 addWorker 失败，向下执行，因为 addWorker 可能改变 ctl 的值，
			// 所以这里重新获取下 ctl
            c = ctl.get();
        }
		
		// 到这步要么是 corePoolSize 满了，要么是 addWorker 失败了
		// 前者很好理解，后者为什么会失败呢？addWorker 中会讲
		
		// 如果线程池状态为 RUNNING 且 task 插入 Queue 成功
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
			// 如果已不处于 RUNNING 状态，那么删除已经入队的 task，然后执行拒绝策略
			// 这里主要是担心并发场景下有别的线程改变了线程池状态，所以 double-check 下
            if (! isRunning(recheck) && remove(command))
                reject(command);
			// 这个分支有点难以理解，意为如果当前 workerCount=0 的话就创建一个线程
			// 那为什么方法开头的那个 addWorker(command, true) 会返回 false 呢，其实
			// 这里有个场景就是 newCachedThreadPool，corePoolSize=0，maxPoolSize=MAX 的场景，
			// 就会进到这个分支，以 maxPoolSize 为界创建临时线程，firstTask=null
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
		// 这个分支很好理解，workQueue 满了那么要根据 maxPoolSize 创建线程了
		// 如果没法创建说明 maxPoolSize 满了，执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

#### addWorker 方法

```java
	// core 表示以 corePoolSize 还是 maxPoolSize 为界
	private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 看看 addWorker 什么时候返回 false
			// 这里的 if 逻辑有点难懂，用下数学上的分配率，将第一个逻辑表达式放进括号里就好懂了
			// 1、rs >= SHUTDOWN && rs != SHUTDOWN 其实就表示当线程池状态是 STOP、TIDYING, 或 TERMINATED 的时候，当然不能添加 worker 了，任务都不执行了还想加 worker？
			// 2、rs >= SHUTDOWN && firstTask != null 表示当提交一个非空任务，但线程池状态已经不是 RUNNING 的时候，当然也不能 addWorker，因为你最多只能执行完 Queue 中已有的任务
			// 3、rs >= SHUTDOWN && workQueue.isEmpty() 如果 Queue 已经空了，那么不允许新增
			// 需要注意的是，如果 rs=SHUTDOWN && firstTask=null 或者 rs=SHUTDOWN && workQueue 非空的情况下，还是可以新增 worker 的，需要创建临时线程处理 Queue 里的任务
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
				// 这里也是一个返回 false 的情况，但很简单，就是数目溢出了
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
				// CAS 成功了，就跳出 loop
                if (compareAndIncrementWorkerCount(c))
                    break retry;
				// CAS 失败的话，check 下目前线程池状态，如果发生改变就回到外层 loop 再来一遍，这个也好理解，否则单纯 CAS 失败但是线程池状态不变的话，就只要继续内层 loop 就行了
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
				// 这是全局锁，必须持有才能进行 addWorker 操作
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
					// 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

#### runWorker 方法

```java
	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
			// 循环直至 task = null，可能是由于线程池关闭、等待超时等
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 下面这个 if 逻辑没怎么读懂。。。翻译了下注释
				// 如果线程池停止，确保线程中断;
				// 如果没有，确保线程不中断。这需要在第二种情况下进行重新获取ctl，以便在清除中断时处理shutdownNow竞争
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
					// 前置钩子函数，可以自定义
                    beforeExecute(wt, task); 
                    Throwable thrown = null;
                    try {
						// 运行 run 方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
						// 线程的 run 不允许抛出 Throwable，所以转换为 Error 
                        thrown = x; throw new Error(x);
                    } finally {
					// 后置钩子函数，也可以自定义
                        afterExecute(task, thrown);
                    }
                } finally {
					// 获取下一个任务
                    task = null;
					// 增加完成的任务数目
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

### 写在最后

看完 `ThreadPoolExecutor` 的源码，不得不惊叹于代码写得真优雅，但是正因为写的太简洁优雅甚至找不到一句啰嗦的代码，所以让人有点难懂。看源码的建议是先仔细阅读一遍类注释，然后再配合 `debug`，理清关键性的步骤在做什么，有些 `corner case` 夹杂在主逻辑里面，如果一开始看不懂可以直接略过，事后再来反思。

### License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>


