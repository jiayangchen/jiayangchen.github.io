---
layout: post
title: "基于 “不可变类” 实现一个线程安全的 Integer 计数器"
subtitle: "不可变对象最核心的地方在于不给外部修改共享资源的机会，从而避免多线程情况下由于争抢共享资源导致的数据不一致"
date: 2019-02-28
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Java 并发编程
---

### 写在前面

众所周知，`java.lang.String` 类没有任何强制同步的代码，但它是线程安全的，原因在于它是不可变类，每次对其操作都是返回一个新的 `String` 对象。合理使用不可变对象可以达到 `lock-free` 的效果。

不可变对象最核心的地方在于**不给外部修改共享资源的机会**，从而避免多线程情况下由于争抢共享资源导致的数据不一致，又是 `lock-free` 的，避免了用锁带来的性能损耗。

### ImmutableIntegerCounter

```java
// final 修饰，不能继承
public final class ImmutableIntegerCounter {

    // final 修饰，不允许其他线程对其更改
    private final int initial;

    public ImmutableIntegerCounter(int initial) {
        this.initial = initial;
    }

    public ImmutableIntegerCounter(ImmutableIntegerCounter counter, int i) {
        this.initial = counter.getInitial() + i;
    }

    // 每次调用 increment 都会新构造一个 ImmutableIntegerCounter
    public ImmutableIntegerCounter increment(int i) {
        return new ImmutableIntegerCounter(this, i);
    }

    public int getInitial() {
        return initial;
    }
}
```

### 测试

```java
import org.junit.Before;
import org.junit.Test;

import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

public class ImmutableIntegerCounterTest {

    private ImmutableIntegerCounter integerCounter = null;

    @Before
    public void init() {
        integerCounter = new ImmutableIntegerCounter(0);
    }

    @Test
    public void incrementTest() throws InterruptedException {

        IntStream.range(0, 3).forEach(i -> new Thread(() -> {
            int increment = 1;
            while(true) {
                int oldValue = integerCounter.getInitial();
                int result = integerCounter.increment(increment).getInitial();
                System.out.println(oldValue + "+" + increment + "=" + result);
                assert (oldValue + increment) == result;
                increment++;
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start());

        TimeUnit.SECONDS.sleep(10);

    }
}

```

### 输出结果

三个线程的输出结果均是正确的

```shell
0+1=1
0+1=1
0+1=1
0+2=2
0+2=2
0+2=2
0+3=3
0+3=3
0+3=3
0+4=4
0+4=4
0+4=4

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

### 许可协议

- 本文遵守创作共享 [CC BY-NC-SA 3.0协议](https://creativecommons.org/licenses/by-nc-sa/3.0/cn/)

