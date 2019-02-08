---
layout: post
title: "回顾《深入理解 Java 虚拟机》之类加载器"
subtitle: ""
date: 2019-02-08
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - JVM
---

虚拟机团队把类加载阶段中的 **“通过一个类的全限定名来获取描述此类的二进制字节流”** 这个动作放到 `Java` 虚拟机外部去实现，以便让应用程序自己去决定如何获取所需要的类，实现这个动作的代码模块称为 **“类加载器”**。

首先，先要知道什么是类加载器。简单说，类加载器就是根据指定全限定名称将 `Class` 文件加载到 `JVM` 内存，转为 `Class` 对象。

如果站在虚拟机的角度来看，只存在两种类加载器:

![](http://ww1.sinaimg.cn/large/c3beb895gy1fzzde8e3udj21bv0do0v6.jpg)

如果从 `Java` 程序员的角度看，还可以进一步细分：

![](http://ww1.sinaimg.cn/large/c3beb895gy1fzzdhqc12wj21g70iyn0w.jpg)

我们的应用程序都是由这三种类加载器互相配合进行加载的，如果有必要还可以加入自定义的类加载器。


### 双亲委派模型

前文所说的那些类加载器的关系如下图：

![](http://ww1.sinaimg.cn/large/c3beb895gy1fzzdnuh4doj20op0p5di9.jpg)

这种层次关系称为类加载器的**双亲委派模型**，其要求除了顶层的启动类加载器之外，其余的类加载器都要有自己的父类加载器（这种父子关系一般通过**组合**来实现，而非继承）。

双亲委派模型工作过程是：如果一个类加载器收到类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器完成。每个类加载器都是如此，只有当父加载器在自己的搜索范围内找不到指定的类时，子加载器才会尝试自己去加载。

### 为什么需要双亲委派模型

有一个显而易见的好处是 `Java` 类随着它的类加载器一起具备了一种带有优先级的层次关系。例如`java.lang.Object`，它存放在 `rt.java` 中，无论哪一个类加载器要加载这个类，最终都会委派给处于模型最顶端的启动类加载器进行加载，因此 `Object` 类在程序的各种类加载器环境中都是同一个类。相反，如果没有双亲委派模型，而是由各个类加载器去自行加载的话，如果用户自己编写了一个 `java.lang.Object` 类，并放在程序的 `ClassPath` 中，那么系统中将会出现多个不同的 `Object` 类，`Java` 体系中最基础的行为也就无法保证，应用程序会变得一片混乱。

### 如何实现双亲委派模型

双亲委派模型的原理很简单，实现也简单。每次通过先委托父类加载器加载，当父类加载器无法加载时，再自己加载。其实ClassLoader类默认的loadClass方法已经帮我们写好了，我们无需去写。

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 首先，检查一下指定名称的类是否已经加载过，
		// 如果加载过了，就不需要再加载，直接返回。
        Class c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
				// 如果此类没有加载过，那么再判断一下是否有父加载器
				// 如果有父加载器，则由父加载器加载
				// 没有父类即是顶层了，就调用启动类加载器来加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 抛出 ClassNotFoundException 说明父类加载器无法完成加载
				// ignore 这个异常
            }
 
            if (c == null) {
                // 如果父加载器或者启动类加载器都没有找到指定的类
				// 那么调用当前类加载器的 findClass 方法来完成类加载
                long t1 = System.nanoTime();
                c = findClass(name);
 
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 参考资料
-《深入理解Java虚拟机》 周志明著

### License
- 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
