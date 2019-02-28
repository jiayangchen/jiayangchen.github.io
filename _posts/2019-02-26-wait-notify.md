---
layout: post
title: "Java 中的 Wait 和 Notify 机制"
subtitle: "Wait 和 Notify 是 Java 面试中常见的问题，但是在平时工作中可能不常见到"
date: 2019-02-26
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java 并发编程 
---

### 写在前面

`Wait` 和 `Notify` 是 `Java` 面试中常见的问题，但是在平时工作中可能不常见到。大家或多或少知道些背景知识，例如二者均为 `Object` 类的方法，而不是 `Thread` 特有的（因为`锁`是每个对象都具有的特性，因此操作锁的方法也紧跟对象，没毛病），且都只能在`同步代码块`中调用（即前提是先获得对象的`监视器锁`，一般来说在 `synchronized` 代码块中使用），否则抛出异常 `IllegalMonitorStateException`。

`Wait` 会挂起自己让出 `CPU` 时间片，并将自身加入锁定对象的 `Wait Set` 中，释放对象的监视器锁`（monitor）`让其他线程可以获得，直到其他线程调用此对象的 `notify( )` 方法或 `notifyAll( )` 方法，自身才能被唤醒（这里有个特殊情况就是 `Wait` 可以增加等待时间）；`Notify` 方法则会释放监视器锁的同时，唤醒对象 `Wait Set` 中等待的线程，顺序是随机的不确定。

### Wait Set

`虚拟机规范`中定义了一个 `Wait Set` 的概念，但至于其具体是什么样的数据结构规范没有强制规定，**意味着不同的厂商可以自行实现**，但不管怎样，线程调用了某个对象的 `Wait` 方法，就会被加入该对象的 `Wait Set` 中

![在这里插入图片描述](http://ww1.sinaimg.cn/large/c3beb895gy1g0k4aqe25tj20lv06gq5l.jpg)

### Demo 代码

下面通过一段 `demo` 来解释 `Wait` 和 `Notify` 的功能

```java
import java.util.concurrent.TimeUnit;

public class WaitNotify {

    public static void main(String[] args) {

        final Object A = new Object();
        final Object B = new Object();

        Thread t1 = new Thread("t1-thread") {
            @Override
            public void run() {
                synchronized (A) {
                    System.out.println(Thread.currentThread().getName() + "拿到 A 的监视器锁");
                    System.out.println(Thread.currentThread().getName() + "尝试获取 B 的监视器锁");
                    try {
                        System.out.println(Thread.currentThread().getName() + "休眠 2s，不释放 A 的监视器锁");
                        TimeUnit.SECONDS.sleep(2);
                        System.out.println(Thread.currentThread().getName() + "挂起自己，释放 A 的监视器锁");
                        A.wait();
                        System.out.println(Thread.currentThread().getName() + "被唤醒，等待获取 B 的监视器锁");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    synchronized (B) {
                        System.out.println(Thread.currentThread().getName() + "拿到 B 的监视器锁");
                        B.notify();
                    }
                }
            }
        };

        Thread t2 = new Thread("t2-thread") {
            @Override
            public void run() {
                synchronized (B) {
                    System.out.println(Thread.currentThread().getName() + "拿到 B 的监视器锁");
                    System.out.println(Thread.currentThread().getName() + "尝试获取 A 的监视器锁");
                    synchronized (A) {
                        System.out.println(Thread.currentThread().getName() + "拿到 A 的监视器锁");
                        try {
                            System.out.println(Thread.currentThread().getName() + "休眠 2s，不释放 A 的监视器锁");
                            TimeUnit.SECONDS.sleep(2);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println(Thread.currentThread().getName() + "挂起自己，释放 A 的监视器锁，唤醒 t0");
                        A.notify();
                    }
                    try {
                        System.out.println(Thread.currentThread().getName() + "休眠 2s，不释放 B 的监视器锁");
                        TimeUnit.SECONDS.sleep(2);
                        System.out.println(Thread.currentThread().getName() + "挂起自己，释放 B 的监视器锁");
                        B.wait();
                        System.out.println(Thread.currentThread().getName() + "被唤醒");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };

        t1.start();
        t2.start();
    }
}
```

### 输出结果

```shell
t1-thread拿到 A 的监视器锁
t2-thread拿到 B 的监视器锁
t1-thread尝试获取 B 的监视器锁
t2-thread尝试获取 A 的监视器锁
t1-thread休眠 2s，不释放 A 的监视器锁
t1-thread挂起自己，释放 A 的监视器锁
t2-thread拿到 A 的监视器锁
t2-thread休眠 2s，不释放 A 的监视器锁
t2-thread挂起自己，释放 A 的监视器锁，唤醒 t0
t2-thread休眠 2s，不释放 B 的监视器锁
t1-thread被唤醒，等待获取 B 的监视器锁
t2-thread挂起自己，释放 B 的监视器锁
t1-thread拿到 B 的监视器锁
t2-thread被唤醒

Process finished with exit code 0
```

### 时序图

![在这里插入图片描述](http://ww1.sinaimg.cn/large/c3beb895ly1g0k34hljphj20fd0mkq4t.jpg)

### 许可协议

- 本文遵守创作共享 [CC BY-NC-SA 3.0协议](https://creativecommons.org/licenses/by-nc-sa/3.0/cn/)


