---
layout: post
title: "LinkedList 源码分析"
subtitle: ""
date: 2017-12-10
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
---

### UML 类图
![](http://o9oomuync.bkt.clouddn.com/linkedlistLinkedlist%E7%B1%BB%E5%9B%BE.png)

我们可以跟 `ArrayList` 的 `UML` 图对比一下，发现由蓝线圈起来的范围，是 ArrayList 也有的（但值得注意的是，LinkedList 没有实现 RandomAccess 接口，这很好理解因为本身链接表就不支持数组那样的 index 随机访问），而 `AbstractSequentialList`、`Queue` 和 `Deque` 这一个抽象类和两个接口，是在 LinkedList 上新增的，那我们惯例，在进入 LinkedList 子类之前，先去拜访一下新增的这三个小伙伴，对于蓝线范围内的接口和抽象类感兴趣的，可以去翻看上一篇关于 ArrayList 的类图结构的分析。

### AbstractSequentialList
#### Introduction
我觉得阅读源码的时候，可以花时间看一看源码抬头的`注释`，虽然较长但是不难看懂，看完可以对整个类的设计和目的有一个全局的把控。

>  This class provides a skeletal implementation of the List interface to minimize the effort required to implement this interface backed by a "sequential access" data store (such as a linked list).  For random access data (such as an array), AbstractList should be used in preference to this class.

这段注释说了，`AbstractSequentialList` 抽象类为接口 List 提供了实现的骨架，减少了实现一个`顺序访问类型`的数据结构，例如链接表的工作量，但对于需要满足随机访问要求的数据结构，请优先考虑 AbstractList，即 AbstractSequentialList 的父类。

#### Methods
##### get()
```java
public E get(int index) {
        try {
            //先由 listIterator(index) 得到值，再由 next 函数返回值并将游标后移
            return listIterator(index).next();
        } catch (NoSuchElementException exc) {
            throw new IndexOutOfBoundsException("Index: "+index);
        }
    }
```

##### set()
```java
public E set(int index, E element) {
        try {
            //拿到指向元素的 ListIterator
            ListIterator<E> e = listIterator(index);
            //拿到值，cursor 后移
            E oldVal = e.next(); 
            //替换值
            e.set(element);
            //返回旧值
            return oldVal;
        } catch (NoSuchElementException exc) {
            throw new IndexOutOfBoundsException("Index: "+index);
        }
    }
```

##### addAll()
```java
//用获取到的 listIterator 逐个添加集合中的元素，得益于 LinkedList 双向链表的特性，每次 add 只需要调整指针指向就可以了
public boolean addAll(int index, Collection<? extends E> c) {
        try {
            boolean modified = false;
            ListIterator<E> e1 = listIterator(index); //获取指定位置起始的迭代器
            Iterator<? extends E> e2 = c.iterator(); 
            while (e2.hasNext()) {
                e1.add(e2.next()); //添加元素
                modified = true;
            }
            return modified;
        } catch (NoSuchElementException exc) {
            throw new IndexOutOfBoundsException("Index: "+index);
        }
    }
```

### Queue
#### Introduction
`Queue` 是一个继承自 `Collection` 的接口，定义和补充了一些 Queue 的特性才有的方法，抬头注释给出了很好的定义：

> A collection designed for holding elements prior to processing.

即是一个设计出来保持`元素次序`的数据结构。

#### Methods
其中`特色`的一些方法有：

```java
boolean offer(E e); //在容量有限的队列中，优先使用 offer 实现插入数据
E poll(); //删除队列的头部并返回数值，若对列为空 return null
E peek(); //取出队列头部的数据不删除，若对列为空 return null
```

### Deque
#### Introduction
> A linear collection that supports element insertion and removal at both ends. 

是一种能支持`两端`进行增删操作的线性集合。

#### Methods
其中特色的一些方法有（有一些 `first-last` 对就简写了）：

```java
void addFirst_Last (E e); //在队列 头部/尾部 插入元素 e
boolean offerFirst_Last (E e); //对于实现限制容量的队列适用
E getFirst_Last (); //在 头部/尾部 取元素，若队列为空，抛异常
E peekFirst_Last (); //在 头部/尾部 取元素，若队列为空，返回 null
boolean removeFirst_LastOccurrence(Object o); //删除队列中 第一次/最后一次 出现的对象
void push(E e); //定义的与栈相关的操作，栈也是基于队列的实现的嘛
E pop();
```

### LinkedList
#### Introduction
好了，介绍完了三个新伙伴，终于迎来了主席嘉宾！欢迎进场！

> Doubly-linked list implementation of the {@code List} and {@code Deque} interfaces.  Implements all optional list operations, and permits all elements (including {@code null}).

读完注释，说明 `LinkedList` 是一个`双向链表`，实现了所有规定的接口函数，允许元素为 `null`。

> Operations that index into the list will traverse the list from the beginning or the end, whichever is closer to the specified index.

索引进入链表的操作将从`前后遍历链表`寻找，以此来接近索引指定的位置。

> Note that this implementation is not synchronized. If multiple threads access a linked list concurrently, and at least one of the threads modifies the list structurally, it must be synchronized externally.

意思是 LinkedList 不是`线程安全`的，如果你需要保证线程安全，要么你在外部使用的时候自己确保，要么可以使用 `Collections.synchronizedList` 来包装你的 LinkedList。

> The iterators returned by this class's {@code iterator} and {@code listIterator} methods are <i>fail-fast</i>

什么叫 `fail-fast`（快速失败）呢？就是一旦在迭代器创建之后，LinkedList 发生了结构上的修改即 `structural modification`（结构上的修改指增删元素，改变元素值不算），除非是迭代器自己的操作，否则任何情况下迭代器均会立即抛异常。作者还在之后的注释中友情提醒，快速失败机制不是一种硬保障，依赖这种异常机制来编码是绝对错误的，它只能用来发现 bug。

#### Node
Node 表示链表内的一个节点，由 LinkedList 内的一个`静态内部类`实现：

```java
private static class Node<E> {
        E item; //元素数值
        Node<E> next; //后置指针
        Node<E> prev; //前置指针

        Node(Node<E> prev, E element, Node<E> next) { //构造器，初始化新节点
            this.item = element; 
            this.next = next;
            this.prev = prev;
        }
    }
```

#### 构造器
LinkedList 提供了`两种`形式的构造器：

```java
//空构造器，一个空的链表
public LinkedList() {
    }

//参数是一个集合 c，之后调用 addAll 方法将 c 中的元素全部添加进创建的链表中
public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

#### Methods
##### linkFirst
linkFirst 和 linkLast 两个方法大同小异，只讲一个。

```java
/**
     * Links e as first element. 
     */
    private void linkFirst(E e) {
        final Node<E> f = first; //保存原本的 first 节点
        final Node<E> newNode = new Node<>(null, e, f); //创建新的节点 e，前置指针指向 null，后置指针指向 f
        first = newNode; //更新 first 节点
        if (f == null)
            last = newNode; //如果 e 身后没有节点，说明整个链表只有一个 e
        else
            f.prev = newNode; //否则的话，由于原本 f 前置指针指向 null，现在需要更新其指向新创建的 e
        size++; //更新链表大小
        modCount++; //用于探测并发操作
    }
```

##### linkBefore

```java
/**
     * Inserts element e before non-null Node succ.
     */
    void linkBefore(E e, Node<E> succ) { //参数是一个待插入节点 e 和一个已知后继节点 succ
        // assert succ != null;
        final Node<E> pred = succ.prev; //获取 succ 的前置节点，e 要插入 prev 和 succ 之间
        final Node<E> newNode = new Node<>(pred, e, succ); //初始化新节点
        succ.prev = newNode; //改变指针指向
        if (pred == null) //因为后继节点肯定存在，但是前置节点不一定，需要判断
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

##### unlink

因为 unlinkFirst 和 unlinkLast 就是 unlink 方法的特殊情况，所以这里只讲 unlink。

```java
/**
     * Unlinks non-null node x.
     */
    E unlink(Node<E> x) { //其实就是删除一个节点
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next; //找出前置后置节点
        final Node<E> prev = x.prev;

        if (prev == null) { //如果待删除的节点就是表头，那么直接让后继成为表头就行了
            first = next;
        } else {
            prev.next = next; //否则的话，让前置的后向指针指向后置节点，跳过待删除的节点
            x.prev = null; //更新待删除的节点的前置指针指向
        }

        if (next == null) { //待删除的是表尾节点，直接更新前置为表尾节点
            last = prev;
        } else {
            next.prev = prev; //与 prev 那边正好相反
            x.next = null;
        }

        x.item = null; //可供 gc 回收
        size--; //更新链表大小
        modCount++;
        return element;
    }
```

##### addAll

```java
public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index); //检查 index 合法性

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ; //定位 index 位置的前置和后置节点
        if (index == size) { 
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode; //前置节点链接新节点
            pred = newNode; //更新循环时的 prev 为最新链接的节点，使得循环能一直进行下去
        }

        //这里注意，因为循环里只是一直链接了前置节点的后置指针，但是当前节点的后置指针指向并未在循环中得到链接，目前还是 null
        if (succ == null) {
            last = pred; //如果没有后继节点了，那自己就不客气了
        } else {
            pred.next = succ; //否则的话，还是老老实实更新指针指向
            succ.prev = pred;
        }

        size += numNew; //更新链表大小
        modCount++;
        return true;
    }
```

##### node
这是用来返回指定位置节点的方法

```java
    Node<E> node(int index) {
        // 看看是前一半还是后一半，决定从前往后找还是从后往前找
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

##### ListItr
一个`私有内部类`，实现`双向`迭代器。

```java
private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned; //最后一次返回的节点
        private Node<E> next; //当前节点
        private int nextIndex; //当前节点 index，不是下一个节点的 index！
        private int expectedModCount = modCount; //用于实现 fail-fast 机制

        ListItr(int index) { //获得 index 位置节点
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() { //返回当前节点值
            checkForComodification(); 
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next; //将当前节点赋给lastReturned，用于返回值
            next = next.next; //指针后移一位
            nextIndex++;
            return lastReturned.item; //返回保存的值
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }

        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned) 
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) { //不用多说了吧
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) { //不用多说了吧
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

### Conclusion
LinkedList 是一个实现了 List 接口和 Deque 接口的双端链表，LinkedList 不是线程安全的。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>