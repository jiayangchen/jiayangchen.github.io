---
layout: post
title: "ArrayList 源码分析"
subtitle: "ArrayList 的源码分析和整体类层结构分析"
date: 2017-12-09
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
    - Java Collections
---

### UML 图
先看一下 `ArrayList` 涉及的 `UML 类图`，从大局上把握一下类结构：

![](http://o9oomuync.bkt.clouddn.com/arraylistArrayList%E7%B1%BB%E5%9B%BE.png)

我们先讲一下整个类图涉及的`接口和抽象类`：
* 最顶层是 `Iterable` 接口，Iterable 意思是可迭代的，用于形容接口很合适。

* 紧接着是 `Collection` 接口，继承自 `Iterable`，并添加了一些集合类的常用函数：`size()`、`isEmpty()`、`contains()`等等，位置处于顶层，因为接下来的实现此接口的类都需要实现这些公共方法，因此一定程度上也起到了`约束规范`的作用。值得一提的是其中的 `equals` 和 `hashcode` 方法，应该大家非常熟悉了

* 接下来，我们来看看继承了 Collection 接口的 `List` 接口，作为更加具体的接口，针对 List 的特性新加入 get、set、sort、sublist 等方法

* 由顶向下我们接口暂时告一段落，`Cloneable`、`Serializable`、`RandomAccess` 这三个主要作为`标识接口`使用，意思是这个类具有怎样的特性，类似于商品打上个`标签`的作用

### AbstractCollection
`AbstractCollection` 作为一个抽象类，做为具体的集合子类的最顶层父类出现的，之后 `AbstractList` 作为具体 List 类别的直接父类出现，如果你从没看过 JDK 集合类的源码，那你应该也可以猜到，其他的例如 AbstractSet、AbstractMap 应该也存在(到底存不存在呢？之后的文章里再说)。这样层级的设计方式，使得类之间各司其职。

`AbstractCollection` 中既有自己实现了方法体的一些函数，例如 `isEmpty`、`contains` 等，也有抽象方法约束着子类去实现，如 `size`。这里指的注意的是，AbstractCollection 不支持单个添加元素的 `add` 方法，会直接报不支持异常，需要子类自己去实现，并且 add 也并没有定义成抽象方法：

![](http://o9oomuync.bkt.clouddn.com/arraylistadd%E6%96%B9%E6%B3%95.png)

这里如果究其原因，我想可能是由于集合类`设计模式`的要求，因为具体的集合子类根据自身的特性不同，实现的方法也会不一样，如果定义成抽象方法，那么之后的子类均需要实现此方法，很繁琐不灵活，现在交给子类自己去实现，相当于把具体的决定权交给了子类，自由度更高。

### AbstractList
接下去是 `AbstractList`，我们已经越来越接近 `ArrayList` 了，作为 `ArrayList` 的直接父类，其实大同小异，这里我们先看`迭代器`的实现吧，
`AbstractList` 中涉及到迭代器的有两个：

```java
private class Itr implements Iterator<E> {}
private class ListItr extends Itr implements ListIterator<E> {}
```
其中 `ListItr` 算是 `Ltr` 的增强版，先看看 Ltr 中的两个主要函数：

![](http://o9oomuync.bkt.clouddn.com/arraylistnext&remove.png)

#### Ltr Next 方法：

1. 先把游标赋给 `i`
2. 拿到游标后的第一个元素 `next`
3. 将 `lastRet` 置为 `i` ，表示最新一次访问的元素位置
4. 游标后移一位
5. 返回数值

#### Ltr Remove 方法：

1. 判断 `lastRet` 是不是 `-1`，如果是的话说明这个元素被一个 `remove` 方法调用了，不能连续调用两次 `remove`，会报错
2. 如果不是 `-1`，紧接着判断并发操作
3. 调用子类实现的 remove 方法删除最新访问的元素，重新更新游标
4. 上次访问的元素位置置为 `-1`

**注意：** 我阅读源码的时候一直很疑惑这个 `lastRet` 真正的意思，到底是标记最新访问的元素的位置，还是跟调用的函数也有关系，最后查阅资料发现，`lastRet` 应该是标记迭代器最后一次调用 `next()` 或者 `previous()` 访问的元素的坐标，当调用 `remove()` 或者 `add()` 操作时（注意：不一定是针对这个元素的操作），该值被设定为 `-1`；

#### ListLtr 
因此，`Ltr` 看下来只是简单实现了 `Iterator` 中的 `next` 和 `remove` 方法，对于一个完整的迭代器类来说远远不够，这时候就需要 `ListLtr` 出马了：

![](http://o9oomuync.bkt.clouddn.com/arraylistprevious&set.png)

`nextIndex` 和 `previousIndex` 都很好理解，不多说了。`previous` 方法中，返回迭代器游标之前的一个元素，并同时设定游标值和最后一次访问元素的坐标值。当调用 `previous` 函数时，可能出现最后一次访问元素坐标与游标重合，其它情况下则不会出现。`Set` 方法则是将列表最后一次访问的元素替换为 `e` 。

#### equals()

然后，我们看看 `AbstractList` 中关于 `equals` 方法的实现：

![](http://o9oomuync.bkt.clouddn.com/arraylistequals.png)

1. 如果是本身直接返回 `true`
2. `instanve of` 判断类型部署 `List` 的话直接 `false`
3. 否则取双发迭代器，依次比较元素，如果其中一个为 `null` 又或者 二者不相等 就 `false`
4. 为啥不先判断长度是否一样呢？

### ArrayList
终于来到 `ArrayList` 了，之前所有的铺垫要到了交出成绩的时刻。

#### 构造器
`ArrayList` 提供三种形式的构造器：

![](http://o9oomuync.bkt.clouddn.com/arraylistconstructor.png)

1. 第一种参数构造器，传入初始大小
2. 第二种无参构造器
3. 第三种参数构造器传入集合

#### 数据域
`ArrayList` 的`数据域`是：

![](http://o9oomuync.bkt.clouddn.com/arraylistelement.png)

> transient 关键字是 Java 语言的关键字，用来表示一个域不是该对象序列化的一部分。当一个对象被序列化的时候，transient 型变量的值不包括在序列化的表示中，然而非 transient 型的变量是被包括进去的。

#### grow()
![](http://o9oomuync.bkt.clouddn.com/arraylistgrow.png)

`grow` 函数表示 ArrayList 的扩容大小，为原先的 `1.5 倍`

#### add()
![](http://o9oomuync.bkt.clouddn.com/arraylistadd1.png)

1. 先检查 `index` 合法性
2. 在确保空间大小不会溢出
3. 为 `Index` 位置的元素腾出空间
4. 最后塞进元素，`size ++`

#### remove()
![](http://o9oomuync.bkt.clouddn.com/arraylistremove.png)

值得一提的是，`remove`、`removeRange` 等方法的实现也差不多，例如 `remove` 中需要压缩列表，将最后一个位置置为 null ，让 `gc` 清理，因为涉及到修改的话，`modCount` 会 ++ ；`fastRemove` 为私有方法，跳过了越界检查并且不返回删除的元素值

#### clone()
![](http://o9oomuync.bkt.clouddn.com/arraylistclone.png)

`Clone` 方法执行`浅复制`，即容器中的元素本身并不复制，与之对应的叫`深复制`，针对集合中的`非基本数据类型`，都需要显示调用 `clone` 方法进行复制。

### 关于 Vector
`Vector` 也是基于数组实现的，是一个动态数组，其容量能自动增长，它的源码实现总体与 `ArrayList` 类似。`Vector` 有四个不同的构造方法。无参构造方法的容量为默认值 `10`，仅包含容量的构造方法则将容量增长量（从源码中可以看出容量增长量的作用，第二点也会对容量增长量详细说）明置为 `0`。`Vector` 内部很多方法都加入了 `synchronized` 关键字，来保证线程安全。目前 `Vector` 已经基本`不再使用`。

### 总结
其他方法就不再细说了，最后总结一下 `ArrayList` 的特性吧：

1. 是一个动态数组，容量可以自动扩容，不适合频繁增加元素的场景
2. 线性存储结构，需要调整、压缩、移动底层数组元素，因此对于增删的时间复杂度较高
3. 可以以索引访问元素，随机访问效率较好
4. 没见到对线程安全的保障，因此并非线程安全的集合类

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>