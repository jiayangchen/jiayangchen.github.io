---
layout: post
title: "【转载】Java 内存泄漏及避免方法"
subtitle: ""
date: 2017-08-01
author: "X Wang"
header-img: "img/db.jpg"
catalog: true
tags: 
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>

> 原文链接： [X Wang](http://www.programcreek.com/2013/10/the-introduction-of-memory-leak-what-why-and-how/) 翻译： ImportNew.com - strongme  译文链接： http://www.importnew.com/12961.html

Java语言的一个关键的优势就是它的内存管理机制。你只管创建对象，Java的垃圾回收器帮你分配以及回收内存。然而，实际的情况并没有那么简单，因为内存泄漏在Java应用程序中还是时有发生的。

下面就解释下什么是内存泄漏，它为什么会发生，以及我们如何阻止它的发生。

### 1. 什么是内存泄漏？

内存泄漏的定义：对象已经没有被应用程序使用，但是垃圾回收器没办法移除它们，因为还在被引用着。

要想理解这个定义，我们需要先了解一下对象在内存中的状态。下面的这张图就解释了什么是无用对象以及什么是未被引用对象。

![image](http://www.programcreek.com/wp-content/uploads/2013/10/where-is-memory-leak-650x400.jpeg)

上面图中可以看出，里面有被引用对象和未被引用对象。未被引用对象会被垃圾回收器回收，而被引用的对象却不会。未被引用的对象当然是不再被使用的对象，因为没有对象再引用它。然而无用对象却不全是未被引用对象。其中还有被引用的。就是这种情况导致了内存泄漏。

### 2. 为什么会发生内存泄漏？

来先看看下面的例子，为什么会发生内存泄漏。下面这个例子中，A对象引用B对象，A对象的生命周期（t1-t4）比B对象的生命周期（t2-t3）长的多。当B对象没有被应用程序使用之后，A对象仍然在引用着B对象。这样，垃圾回收器就没办法将B对象从内存中移除，从而导致内存问题，因为如果A引用更多这样的对象，那将有更多的未被引用对象存在，并消耗内存空间。

B对象也可能会持有许多其他的对象，那这些对象同样也不会被垃圾回收器回收。所有这些没在使用的对象将持续的消耗之前分配的内存空间。

![image](http://www.programcreek.com/wp-content/uploads/2013/10/object-life-time-650x508.jpeg)

### 3. 如何防止内存泄漏的发生？

下面是几条容易上手的建议，来帮助你防止内存泄漏的发生。

1. 特别注意一些像HashMap、ArrayList的集合对象，它们经常会引发内存泄漏。当它们被声明为static时，它们的生命周期就会和应用程序一样长。

2. 特别注意事件监听和回调函数。当一个监听器在使用的时候被注册，但不再使用之后却未被反注册。

3. “如果一个类自己管理内存，那开发人员就得小心内存泄漏问题了。” 通常一些成员变量引用其他对象，初始化的时候需要置空。

### 4. 一个小问题：为什么JDK6中的substirng()方法容易导致内存泄漏？

#### Java 6中substring的实现

```java
public String substring(int beginIndex, int endIndex) {
  if (beginIndex < 0) {
      throw new StringIndexOutOfBoundsException(beginIndex);
  }
  if (endIndex > count) {
      throw new StringIndexOutOfBoundsException(endIndex);
  }
  if (beginIndex > endIndex) {
      throw new StringIndexOutOfBoundsException(endIndex - beginIndex);
  }
  return ((beginIndex == 0) && (endIndex == count)) ? this :
      new String(offset + beginIndex, endIndex - beginIndex, value);
}
```

上述方法调用的构造方法

```java
//Package private constructor which shares value array for speed.
String(int offset, int count, char value[]) {
  this.value = value;
  this.offset = offset;
  this.count = count;
}
```

当我们调用字符串a的substring得到字符串b，其实这个操作，无非就是调整了一下b的offset和count，用到的内容还是a之前的value字符数组，并没有重新创建新的专属于b的内容字符数组。

举个和上面重现代码相关的例子，比如我们有一个1G的字符串a，我们使用substring(0,2)得到了一个只有两个字符的字符串b，如果b的生命周期要长于a或者手动设置a为null，当垃圾回收进行后，a被回收掉，b没有回收掉，那么这1G的内存占用依旧存在，因为b持有这1G大小的字符数组的引用。

看到这里，大家应该可以明白上面的代码为什么出现内存溢出了。

#### 共享内容字符数组

其实substring中生成的字符串与原字符串共享内容数组是一个很棒的设计，这样避免了每次进行substring重新进行字符数组复制。正如其文档说明的,共享内容字符数组为了就是速度。但是对于本例中的问题，共享内容字符数组显得有点蹩脚。

#### 如何解决

对于之前比较不常见的1G字符串只截取2个字符的情况可以使用下面的代码，这样的话，就不会持有1G字符串的内容数组引用了。

```java
String littleString = new String(largeString.substring(0,2));
```

下面的这个构造方法，在源字符串内容数组长度大于字符串长度时，进行数组复制，新的字符串会创建一个只包含源字符串内容的字符数组。

#### Java 7 实现

在Java 7 中substring的实现抛弃了之前的内容字符数组共享的机制，对于子字符串（自身除外）采用了数组复制实现单个字符串持有自己的应该拥有的内容。

```java
public String substring(int beginIndex, int endIndex) {
    if (beginIndex < 0) {
      throw new StringIndexOutOfBoundsException(beginIndex);
    }
    if (endIndex > value.length) {
      throw new StringIndexOutOfBoundsException(endIndex);
    }
    int subLen = endIndex - beginIndex;
    if (subLen < 0) {
      throw new StringIndexOutOfBoundsException(subLen);
    }
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
}
```

substring方法中调用的构造方法，进行内容字符数组复制。

```java
public String(char value[], int offset, int count) {
    if (offset < 0) {
          throw new StringIndexOutOfBoundsException(offset);
    }
    if (count < 0) {
      throw new StringIndexOutOfBoundsException(count);
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
      throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```

