---
layout: post
title: "深入理解java虚拟机 —— java 内存模型"
subtitle: "最近在看周志明的《深入理解Java虚拟机》，将其中的内容总结下来写几篇总结好了。"
date: 2017-04-14
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 深入理解Java虚拟机
    - 面试
---
<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@emilep?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Émile Perron"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Émile Perron</span></a>

最近在看周志明的《深入理解Java虚拟机》，将其中的内容总结下来写几篇总结好了。

Java虚拟机在执行Java程序的时候会把它所管理的内存分为若干个不同的数据区域，如下图：
![image](http://o9oomuync.bkt.clouddn.com/Java%20%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA.png)

### 程序计数器
程序计数器所需内存较小，可看作是指向当前线程所执行的字节码的行号指示器，字节码解释器工作时通过改变PC的值选择下一条字节码指令。
每个线程有自己独立的程序计数器，这样在多线程切换的情况下各线程的PC互不影响，独立储存。
如果线程正在执行Java方法，则程序计数器记录的是当前字节码指令的地址，若是Native方法，则为空。

### Java虚拟机栈
也是线程私有的，其生命周期与线程相同，每个方法在执行的时候都会创建栈帧存储局部变量表、操作数栈、动态链接、方法出口等信息，每个方法从调用到完成的过程，就对应着一个栈帧在JVM栈中入栈到出栈的过程。
值得注意的是，局部变量表存储编译期可知的各种基础数据类型、对象引用加上 returnAddress 类型。其中局部变量表的大小在编译期间分配完成，在方法运行期间不会改变局部变量表的大小。

### 本地方法栈
与JVM栈相似，只不过本地方法栈为Java所使用到的Native方法服务，HotSpot虚拟机直接将本地方法栈和虚拟机栈合二为一。

### Java堆
java虚拟机管理的内存中最大的一块，被所有线程共享的一块内存区域，唯一目的是存放对象实例，几乎所有实例都在这分配内存。
Java堆是GC主要负责的区域，可分为新生代、老年代；更仔细一点有Eden、From Survive、To Survive、Old。值得注意的是，从内存分配的角度看，Java堆中可能划分出多个线程私有的分配缓冲区（TLAB）

### 方法区
线程共享，储存已被虚拟机加载的类信息、常量、静态变量、即时编译后的代码等数据。

#### 运行时常量池
方法区的一部分，用于存放编译期生成的各种字面量和符号引用

### 对象访问
对象访问在Java语言中无处不在，即使是最简单的访问，也会涉及到Java栈，java堆，方法区这三个最重要的内存区域之间的关联关系。如下面的代码：
```java
       Object obj = new Object();
```
假设这段代码出现在方法体中，那么“Object obj”部分的语义将会反映到Java栈的本地变量表中，作为一个reference类型的数据存在。而“new Object();”部分的语义将会反应到Java堆中，形成一块存储Object类型所有实例数据值（Instance Data）的结构化内存，根据具体类型以及虚拟机实现的对象分布的不同，这块内存的长度是不固定的。另外，在JAVA堆中还必须包含能查找到此对象内存数据的地址信息，这些类型数据则存储在方法区中。

由于reference类型在Java虚拟机中之规定了指向对象的引用，并没有规定这个引用要通过哪种方式去定位，以及访问到Java堆中的对象的具体位置，因此虚拟机实现的对象访问方式会有所不同。主流的访问方式有两种：句柄访问方式和直接指针。

* 1. 如果使用句柄访问方式，Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的地址，而句柄中包含了对象实例数据和类型数据各自的具体地址信息。

![image](http://img.blog.csdn.net/20141116180521750?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvT3lhbmdZdWp1bg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

* 2. 如果通过直接指针方式访问，Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，reference中直接存储的就是对象的地址。

![image](http://img.blog.csdn.net/20141116180738062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvT3lhbmdZdWp1bg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

两种方式各有优势，局并访问方式最大的好处是reference中存放的是稳定的句柄地址，在对象被移动时，只会改变句柄中的实例数据指针，而reference本身不需要被修改。而指针访问的最大优势是速度快，它节省了一次指针定位的开销，由于对象访问在Java中非常频繁，一次这类开销积少成多后也是一项非常可观的成本。具体的访问方式都是有虚拟机指定的，虚拟机Sun HotSpot使用的是直接指针方式，不过从整个软件开发的范围来看，各种语言和框架使用句柄访问方式的情况十分常见。

### 参考资料
* 《深入理解Java虚拟机》 周志明著