---
layout: post
title: "PriorityQueue 源码分析"
subtitle: "PriorityQueue 的源码分析和整体类层结构分析"
date: 2017-12-19
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
---

### UML diagram

首先，我们来规矩来看看 `PriorityQueue` 的 UML 类图，被`蓝线`圈起来的部分是我们第一次接触到的新朋友，蓝线之外的接口或抽象类都在之前的文章中涉及到过了，感兴趣的可以翻看前几篇文章。

![](http://o9oomuync.bkt.clouddn.com/priorityqueuepriorityQueue%E7%B1%BB%E5%9B%BE.png)

### AbstractQueue

先来看看 `AbstractQueue` 抽象类，继承自 `AbstractCollection`，并且实现了 `Queue` 接口。内部构造非常简单，提供了一个 `protected` 修饰的无参构造器给子类实现，其他还有 `add`、`remove`、`element`、`clear` 和 `addAll` 方法，其方法体内调用的方法大都来自 `Queue` 接口中的定义，尚未实现。

### PriorityQueue
#### Introduction

什么叫 `PriorityQueue` 呢？答曰：优先级队列，结合现实生活也很好理解，你去银行取钱自然要排队，依次取钱；但是手持 `VIP` 卡的人就能进入快速通道比你先取到钱，即使人家来的比你晚。因此，可以猜到，优先级队列必定比普通队列多上至少一个指标，表示其优先级高低，然后以此来决定它在队列中的位置。

> An unbounded priority {@linkplain Queue queue} based on a priority heap. The elements of the priority queue are ordered according to their {@linkplain Comparable natural ordering}, or by a {@link Comparator} provided at queue construction time, depending on which constructor is used. 

Java 中优先级队列是用堆来实现的，可以传入 `comparator` 来自定义比较实现。

> The <em>head</em> of this queue is the <em>least</em> element with respect to the specified ordering.  If multiple elements are tied for least value, the head is one of those elements

其中队首的是最小的元素，如果最小的元素有多个，则取其一。

> A priority queue is unbounded, but has an internal <i>capacity</i> governing the size of an array used to store the elements on the queue.  It is always at least as large as the queue size.  As elements are added to a priority queue, its capacity grows automatically.

优先级队列是无边界的，但是通过 `capacity` 约束实际数组的大小，最起码跟 `size` 一样大，否则会扩容。

> Note that this implementation is not synchronized.</strong> Multiple threads should not access a {@code PriorityQueue} instance concurrently if any of the threads modifies the queue.

很显然，`不是线程安全`的，多线程并发环境会出问题，想要保持线程安全可以外部实现或者采用线程安全类。

#### Attribute

```java
private static final int DEFAULT_INITIAL_CAPACITY = 11; //初始化容量，为什么偏偏是 11 。。。有什么讲究么，求解释

transient Object[] queue; //存放元素的数组，非私有方便嵌套类访问

private int size = 0; //元素个数

private final Comparator<? super E> comparator; //翻译成比较器？就是用于比较你 VIP 级别的高低

transient int modCount = 0; //实现 fast-fail 机制的
```

#### constructor

令人惊奇的是 PriorityQueue 提供了多达 `7` 种构造器，我们依次来看看为什么要有这么多？分别提供什么功能？各有什么好处？

```java
    //默认无参构造器，采用初始化容量 11，无 comparator
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    //只传入指定的初始化容量，不传入 comparator，话说不判断一下是否溢出的么？
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    //只传入 comparator，采用默认容量 11
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    //传入用户指定的初始化容量和 comparator
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)                 //做一个合法性校验
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];//初始化数组
        this.comparator = comparator;            //赋值 comparator
    }

    //传入一个集合
    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) { //是不是 sortedset 类型？
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c; //强制类型转换
            this.comparator = (Comparator<? super E>) ss.comparator(); //获取其 comparator
            initElementsFromCollection(ss); //调用 initElementsFromCollection 方法，含义很明显
        }
        else if (c instanceof PriorityQueue<?>) { //同上
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null; //上述二者都不是的话，comparator 赋值为 null，可见仅有上述二者存在比较器
            initFromCollection(c);
        }
    }

    //传入一个 PriorityQueue 类型的集合
    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    //传入 SortedSet 类型的集合
    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
```

#### methods

##### constructor related

```java
    private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        if (c.getClass() == PriorityQueue.class) { //判断类型
            this.queue = c.toArray(); //直接拿到 Queue 和 size 的值
            this.size = c.size();
        } else {
            initFromCollection(c);
        }
    }

    private void initElementsFromCollection(Collection<? extends E> c) {
        Object[] a = c.toArray(); //集合中的数组内容
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        int len = a.length;
        if (len == 1 || this.comparator != null) //为什么不是所有情况都检测呢？难道其他类里有检测逻辑了？
            for (int i = 0; i < len; i++)
                if (a[i] == null) //检查是否有空元素
                    throw new NullPointerException();
        this.queue = a;
        this.size = a.length;
    }

    private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify(); //建堆
    }
```

##### grow & hugeCapacity

同样的，`PriorityQueue` 中也有扩容方法和判断最大容量的方法，实现也大同小异，我们就粗看看吧。

```java
    private void grow(int minCapacity) {
        int oldCapacity = queue.length; //得到现在队列数组的长度
        // Double size if small; else grow by 50%
        int newCapacity = oldCapacity + ((oldCapacity < 64) ? //如果当前长度小于 64 的，扩容只 +2； 
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1)); //否则扩容一半，即变为原先的 1.5 倍
        // overflow-conscious code
        if (newCapacity - MAX_ARRAY_SIZE > 0) //如果扩容之后的大小以及超出，那就调用 hugeCapacity 方法指定正确的容量
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity); //之后就是转移数组内容了
    }

    //不再啰嗦了
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

##### add & offer

```java
    public boolean add(E e) {
        return offer(e);        //直接调用 offer 方法插入数据
    }

    public boolean offer(E e) {
        if (e == null)          //如果需要插入的数据为 null 抛异常
            throw new NullPointerException();
        modCount++;             //否则记录此次修改
        int i = size;           //拿到当前的元素个数
        if (i >= queue.length)  //看看队列是不是满了
            grow(i + 1);        //满了的话扩容
        size = i + 1;           //个数 ++
        if (i == 0)             //如果队列里尚且没有元素
            queue[0] = e;       //插进首部，其实就是堆的根节点
        else
            siftUp(i, e);       //否则调用 siftUp 方法插入，涉及堆的插入操作
        return true;
    }
```

##### poll

```java
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;          //size 减一
        modCount++;
        E result = (E) queue[0]; //拿出队列头部
        E x = (E) queue[s];      //拿出队列尾部
        queue[s] = null;         //删掉队尾
        if (s != 0)
            siftDown(0, x);      //将队尾元素放在堆的根节点上，然后调整堆结构
        return result;
    }
```

##### removeAt

```java
    private E removeAt(int i) {
        // assert i >= 0 && i < size;
        modCount++;
        int s = --size;
        if (s == i) // removed last element
            queue[i] = null;            //删除队尾，直接置 null
        else {
            E moved = (E) queue[s];     //否则先保存队尾元素
            queue[s] = null;            //删掉队尾
            siftDown(i, moved);         //将队尾元素与待删除的元素交换位置，然后调整堆结构
            if (queue[i] == moved) {
                siftUp(i, moved);
                if (queue[i] != moved) 
                    return moved;
            }
        }
        return null;
    }
```

##### iterator

跟 `ArrayList`，`LinkedList` 等的迭代器实现类似，这里关注 `forgetMeNot` 的存在意义。

这里重点说一下，首先，我们简单的设想一下，一个堆生成了一个迭代器，但是堆的元素是变化的。那这个堆如何保证可以遍历所有元素呢？java里使用的是这样的策略，正常情况下都是按照数组的顺序从 0 到 size 递增读取当然没有问题。但是如果迭代器在迭代的过程中删除了元素。则需要讨论一些特殊的情况。删除元素的过程如上小结所述，如果最末未元素进行了下滤操作，则不需要考虑因为，迭代器是按照顺序进行的。下滤的元素必然会被之后的迭代器迭代到。但如果原最末未元素进行了上移。这需要用另一个堆来保存它。因为上滤的元素不会被之后的迭代器迭代到。

有了这个 `forgetMeNot`，则可以保证迭代器可以迭代所有元素。当迭代器的 `cursor == size` 的时候则说明除去上滤的元素所有的元素都已经进行了迭代。那么下一次 `next` 和 `remove` 的操作则需要从 forgetMeNot 里的元素进行。 `lastRetElt` 则是为了保证必须先进行 `next` 才能进行 `remove`。

```java
    public Iterator<E> iterator() {
        return new Itr();
    }

    private final class Itr implements Iterator<E> {
        
        private int cursor = 0;

        private int lastRet = -1; //记录最近一次返回元素的索引

        private ArrayDeque<E> forgetMeNot = null; //这个数组表示迭代过程中遗漏的元素

        private E lastRetElt = null; //下一个遗漏的元素

        private int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < size ||
                (forgetMeNot != null && !forgetMeNot.isEmpty());
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            if (cursor < size)
                return (E) queue[lastRet = cursor++];
            //运行到这里说明除了在删除过程中上滤的元素外所有元素都已经迭代完了。那就需要查看forgetMeNot 里还有没有需要迭代的元素。
            if (forgetMeNot != null) { 
                lastRet = -1;
                lastRetElt = forgetMeNot.poll();
                if (lastRetElt != null)
                    return lastRetElt;
            }
            throw new NoSuchElementException();
        }

        public void remove() {
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            if (lastRet != -1) {
                E moved = PriorityQueue.this.removeAt(lastRet);
                lastRet = -1;
                if (moved == null) //返回null说明么有进行上滤操作
                    cursor--;
                else {
                    //如果进行了上滤操作则需要将上滤的元素加入forgetMeNot ，以保证所有元素都可以被迭代器遍历。
                    if (forgetMeNot == null) 
                        forgetMeNot = new ArrayDeque<>();
                    forgetMeNot.add(moved);
                }
            } 
            //删除上滤的元素。并将lastRetElt 置为null使得，下次必须先调用next才能进行删除。
            else if (lastRetElt != null) {
                PriorityQueue.this.removeEq(lastRetElt);
                lastRetElt = null;
            } else {
                throw new IllegalStateException();
            }
            expectedModCount = modCount;
        }
    }
```

##### siftUp

`上移`操作

```java
    private void siftUp(int k, E x) {
        if (comparator != null)
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
    }

    //有比较器的时候用这个
    @SuppressWarnings("unchecked")
    private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            int parent = (k - 1) >>> 1; //找到当前节点 k 的父节点，在 k/2 - 1 的位置
            Object e = queue[parent]; //取父节点
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e; //如果父节点小于它，交换，继续判断父节点是否需要上移，终止条件为 k = 0
            k = parent;
        }
        queue[k] = x;
    }

    //没有比较器的时候用这个，类似不啰嗦
    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```

##### siftDown

`下溢`操作

```java
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }

    //要考虑两个节点
    @SuppressWarnings("unchecked")
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;          //先取一半
        while (k < half) {
            int child = (k << 1) + 1;   //先取 2k + 1 处的左子节点
            Object c = queue[child]; 
            int right = child + 1;      //取右子节点
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0) //比较左右节点找出较小的一个
                c = queue[child = right]; 
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;//作为选定的child，像上移操作一样进行循环判断较小的子节点和父节点的大小
            k = child;
        } //终止条件是k已经是叶节点或者k小于其所有子节点
        queue[k] = x;
    }

    @SuppressWarnings("unchecked")
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo((E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
```

##### heapify

`建堆`操作

```java
    private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--) //对非叶节点进行排序
            siftDown(i, (E) queue[i]);  
    }
```

### Conclusion

`PriorityQueue` 实现的方式采用`二叉堆`，将内部的数组视为一棵树，这棵树有这样的一些特性：

* 对应任意节点 `i`，其左子节点为 `( 2 * i + 1 )`、右节点为 `( 2 *  i + 2 )`;
* 数组的前半部分必为非叶节点。

实现了 `siftUp` 和 `siftDown` 两种私有访问的方法用于进行上移和下滤操作。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>