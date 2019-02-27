---
layout: post
title: "用 Wait 和 Notify 实现 “生产者-消费者” 模型"
subtitle: "在实际开发过程中，经常会碰到如下场景：某个模块负责产生数据，然后经由某个共享的缓冲区进行流转，最后这些数据由另一个模块来负责处理"
date: 2019-02-27
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java 并发编程
---

### 写在前面

在实际开发过程中，经常会碰到如下场景：某个模块负责产生数据，然后经由某个共享的缓冲区进行流转，最后这些数据由另一个模块来负责处理。产生数据的模块，就形象地称为生产者；而处理数据的模块，就称为消费者，中间的仓库可以抽象成任何缓冲区类型的存储介质。

那么这种模型有什么优点呢？第一是解耦，可以将生产者和消费者分隔开，耦合度降低；二是支持并发，生产者把制造出来的数据往缓冲区一丢，就可以再去生产下一个数据，基本上不用依赖消费者的处理速度；三是可以协调生产者和消费者的速度差异，不需要强制二者速率一致。

### 生产者-消费者模型

```java
import java.util.LinkedList;
import java.util.concurrent.TimeUnit;

public class ProducerConsumer {

    public static void main(String[] args) throws InterruptedException {

        final int maxSize = 3;
        final LinkedList<String> queue = new LinkedList<String>();

        final Thread producer = new Thread("producer-thread") {
            @Override
            public void run() {
                while(true) {
                    synchronized (queue) {
                        while (queue.size() == maxSize) {
                            try {
                                System.out.println(Thread.currentThread().getName() + "因队列满容挂起自己,让 consumer 开始执行");
                                queue.wait();
                                System.out.println(Thread.currentThread().getName() + "被唤醒，往队列中添加元素");
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        int index = queue.size();
                        queue.add("task-" + index);
                        System.out.println(Thread.currentThread().getName() + "添加任务 task-" + index);
                        queue.notifyAll();
                    }
                }
            }
        };

        Thread consumer = new Thread("consumer-thread") {
            @Override
            public void run() {
                while (true) {
                    synchronized (queue) {
                        while (queue.isEmpty()) {
                            System.out.println(Thread.currentThread().getName() + "因队列为空挂起自己，让 producer 开始执行");
                            try {
                                queue.wait();
                                System.out.println(Thread.currentThread().getName() + "被唤醒，消费队列中的元素");
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println(Thread.currentThread().getName() + "执行任务" + queue.poll());
                        queue.notifyAll();
                    }
                }
            }
        };

        producer.start();
        TimeUnit.MILLISECONDS.sleep(1); // 让 producer 先启动
        consumer.start();

    }

}
```

### 输出结果

```shell
producer-thread添加任务 task-0
producer-thread添加任务 task-1
producer-thread添加任务 task-2
producer-thread因队列满容挂起自己,让 consumer 开始执行
consumer-thread执行任务task-0
consumer-thread执行任务task-1
consumer-thread执行任务task-2
consumer-thread因队列为空挂起自己，让 producer 开始执行
producer-thread被唤醒，往队列中添加元素
producer-thread添加任务 task-0
producer-thread添加任务 task-1
producer-thread添加任务 task-2
producer-thread因队列满容挂起自己,让 consumer 开始执行
consumer-thread被唤醒，消费队列中的元素
consumer-thread执行任务task-0
consumer-thread执行任务task-1
consumer-thread执行任务task-2
consumer-thread因队列为空挂起自己，让 producer 开始执行
producer-thread被唤醒，往队列中添加元素
producer-thread添加任务 task-0
producer-thread添加任务 task-1
producer-thread添加任务 task-2
producer-thread因队列满容挂起自己,让 consumer 开始执行
consumer-thread被唤醒，消费队列中的元素
consumer-thread执行任务task-0
consumer-thread执行任务task-1
consumer-thread执行任务task-2
consumer-thread因队列为空挂起自己，让 producer 开始执行
producer-thread被唤醒，往队列中添加元素
producer-thread添加任务 task-0
producer-thread添加任务 task-1

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

### 许可协议

- 本文遵守创作共享 [CC BY-NC-SA 3.0协议](https://creativecommons.org/licenses/by-nc-sa/3.0/cn/)
