---
layout: post
title: "HashMap 源码分析"
subtitle: "长文慎入，HashMap 的源码分析和整体类层结构分析"
date: 2017-12-11
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - JDK 源码分析
---

### UML 类图
![](http://o9oomuync.bkt.clouddn.com/hashmaphashmap%E7%B1%BB%E5%9B%BE.png)

不知道大家还记不记得在 `ArrayList` 那篇文章中，我谈到说不定存在 `AbstractSet`、`AbstractMap` 等抽象类的事情，那是基于对类层设计的猜想，现在看到 `HashMap` 的层级类图之后，我们会发现确实存在 `AbstractMap` 这个抽象类，也印证了整个 Java 集合类的设计确实遵循严格的规范，这是值得我们仔细体会和学习的。

### Map
老规矩，我们分析 `HashMap` 之前还是先来看看其他的接口和抽象类，先是最顶层的 `Map` 接口。

#### Introduction

> An object that maps keys to values.  A map cannot contain duplicate keys; each key can map to at most one value.

Map 是一种存储`键值对`的对象，其中键值 key 不允许重复，目的就是一个 key 最多只能对应一个值。

> This interface takes the place of the <tt>Dictionary</tt> class, which was a totally abstract class rather than an interface.

由此可见，Map 接口是取代了老式的 `Dictionary` 抽象类，由接口实现更显轻量、灵活，抽象类会有很多制约，例如不允许多重继承、强制子类实现抽象方法等。

> The <tt>Map</tt> interface provides three <i>collection views</i>, which allow a map's contents to be viewed as a set of keys, collection of values, or set of key-value mappings.  The <i>order</i> of a map is defined as the order in which the iterators on the map's collection views return their elements.  Some map implementations, like the <tt>TreeMap</tt> class, make specific guarantees as to their order; others, like the <tt>HashMap</tt> class, do not.

这段说明了 Map 提供了三种形式的集合视图：

1. keys 的集合
2. values 的集合
3. <key,value> 对的集合

并且，`TreeMap` 是保证插入顺序的，`HashMap` 则并不保证。

#### Methods
```java
    int size(); //返回 map 键值对数目，如果超出了 Integer.MAX_VALUE 则返回 Integer.MAX_VALUE
    boolean containsKey(Object key); // key==null ? k==null : key.equals(k) 最多支持一个 key 为 null
    Set<K> keySet(); //值得注意的是，如果迭代过程中 map 被修改了（除却迭代器自身的修改行为），那么返回的结果是 undefined
    Set<Map.Entry<K, V>> entrySet(); //返回 map 中包含的键值对
```

### AbstractMap
#### Introduction

注释内容和之前的 `AbstractList` 近似，不再啰嗦了，我们看看其中包含的方法吧。

#### Methods
##### entrySet()
```java
    public abstract Set<Entry<K,V>> entrySet();  //作为 AbstractMap 中唯一的抽象方法，返回键值对集合
```

##### get()
```java
    public V get(Object key) {
        Iterator<Entry<K,V>> i = entrySet().iterator(); 
        if (key==null) {
            while (i.hasNext()) { 
                Entry<K,V> e = i.next();
                if (e.getKey()==null) //循环到 key == null 时取出 value
                    return e.getValue();
            }
        } else {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey())) //找到 key 时返回值
                    return e.getValue();
            }
        }
        return null;
    }
```

##### put()
```java
    public V put(K key, V value) {
        throw new UnsupportedOperationException(); //默认不支持 put，子类需要重写 put 方法以实现可变的哈希表
    }
```

##### remove()
```java
    public V remove(Object key) {
        Iterator<Entry<K,V>> i = entrySet().iterator();
        Entry<K,V> correctEntry = null;
        if (key==null) {
            while (correctEntry==null && i.hasNext()) {
                Entry<K,V> e = i.next();
                if (e.getKey()==null)
                    correctEntry = e;
            }
        } else {
            while (correctEntry==null && i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey()))
                    correctEntry = e;
            }
        }

        V oldValue = null;
        if (correctEntry !=null) { //确保找到 entry，否则返回 null
            oldValue = correctEntry.getValue(); //返回旧值
            i.remove(); //调用 Iterator remove 方法删除
        }
        return oldValue;
    }
```

##### equals()
```java
    public boolean equals(Object o) {
        if (o == this) //如果是自身返回 true
            return true;

        if (!(o instanceof Map)) //如果不是 map 的实现类
            return false;
        Map<?,?> m = (Map<?,?>) o; //强制转换成 map
        if (m.size() != size()) //判断数量是否相等
            return false;

        //然后还需要一个个比对
        try {
            Iterator<Entry<K,V>> i = entrySet().iterator();
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                if (value == null) {
                    if (!(m.get(key)==null && m.containsKey(key))) //对于 null 的 key 值需要既判断 key 存在并且值相等
                        return false;
                } else {
                    if (!value.equals(m.get(key)))
                        return false;
                }
            }
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }

        return true;
    }
```

##### hashcode()
```java
// Map<K,V> 的 hash 值为每个映射的 hash 值的总和  
    public int hashCode() {
        int h = 0;
        Iterator<Entry<K,V>> i = entrySet().iterator();
        while (i.hasNext())
            h += i.next().hashCode();
        return h;
    }
```

##### clone()
```java
//万年不变还是浅复制
    protected Object clone() throws CloneNotSupportedException {
        AbstractMap<?,?> result = (AbstractMap<?,?>)super.clone();
        result.keySet = null;
        result.values = null;
        return result;
    }
```

##### SimpleEntry<K,V>
```java
    public static class SimpleEntry<K,V>
        implements Entry<K,V>, java.io.Serializable //SimpleEntry 类，用于实现自定义映射
```

##### SimpleImmutableEntry<K,V>
```java
    public static class SimpleImmutableEntry<K,V>
        implements Entry<K,V>, java.io.Serializable //不可变的 SimpleEntry 类，其内部 put 方法未实现，直接抛异常
```

### HashMap
#### Introduction

>The <tt>HashMap</tt> class is roughly equivalent to <tt>Hashtable</tt>, except that it is unsynchronized and permits nulls.

`HashMap` 和 `HashTable` 基本相似，不过前者不是线程安全的，并且支持 key 和 value 都可以是 null，但 key 为 null 的个数`最多一个`。

> Thus, it's very important not to set the initial capacity too high (or the load factor too low) if iteration performance is important.

如果你对`迭代效率`要求很高的话，要注意设置初始容量大小，不能太大。接下去讲了一些影响 HashMap 性能的两大要素：`初始大小`和`负载因子`。如果你的 HashMap 要存储的数据量很大，那么设置一个较高的初始容量大小比让它频繁扩容触发 rehash 方法要好得多；负载因子需要考虑实际需求，一般默认的 `0.75` 是在时间-空间上权衡下来较好的选择。

> The iterators returned by all of this class's "collection view methods" are <i>fail-fast</i>

迭代器是`快速失败`的，关于什么叫快速失败可以看上一篇 LinkedList 源码分析中的翻译。

#### Attributes

属性值大都采用 static final 的形式定义为不可变的静态常量。

1. 缺省容量（`DEFAULT_INITIAL_CAPACITY`）大小为 16，没有采用直接赋值而是以 `1<<4` 的形式，速度快？数据安全性更好？
2. 最大容量（`MAXIMUM_CAPACITY`）为 1<<30，即 `2^30`
3. 默认负载因子（`DEFAULT_LOAD_FACTOR`）大小为 `0.75f`
4. `TREEIFY_THRESHOLD` 意为一个阈值，超过此值之后采用将后挂链表转换为红黑树的方法解决冲突
5. `UNTREEIFY_THRESHOLD` 也是一个阈值，跟第四条更好相反，意思是在 `resize` 的过程中，当 `bucket` 中的数量小于此值时用链表代替红黑树
6. `MIN_TREEIFY_CAPACITY` 当 `bucket` 中的节点被转化成红黑树时最小的 `hash` 表容量。

#### Structure
在我们分析具体的方法之前，先就 HashMap 整体的结构做一个叙述，简单来说，一个 HashMap 主要由`数组`（即 Bucket）、`链表`、`红黑树`这三者构成，其中链表和红黑树在适当的时候会相互转换。

> 图片来自美团点评技术博客

![](http://tech.meituan.com/img/java-hashmap/hashMap%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)

##### Node<K,V>
```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> { //实现了 Entry 接口
        final int hash; //哈希值，int 类型，final 修饰不可变
        final K key;    //key 值同样由 final 修饰
        V value;        //存储的实际 value 可变
        Node<K,V> next; //指向下一个 next

        Node(int hash, K key, V value, Node<K,V> next) { //构造函数
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        //gettter & setter 方法，这里可以注意一下这个 toString 方法的隔离方式，用等号隔开KV，这个特性有的场景下挺有用的，例如我上次做的一个很复杂的报表，有这个分割方式取值快多了
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        //这个方法应该都不陌生，看看这么做的，是通过获取 key value 二者的哈希值再进行异或操作
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this) //如果是本身的话返回 true
                return true;
            if (o instanceof Map.Entry) { //否则一定要是 Map 的实现
                Map.Entry<?,?> e = (Map.Entry<?,?>)o; //强制类型转换
                if (Objects.equals(key, e.getKey()) && //分别比较 key 和 value 的值
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

#### hash()
将`高 16` 位和`低 16` 位进行`异或`计算，主要是从速度、功效、质量来考虑的，这么做可以在数组 table 的 length 比较小的时候，也能保证考虑到高低 Bit 都参与到哈希值的计算中，同时不会有太大的性能损耗和开销。

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); //将高 16 位加入哈希值计算
    }
```

> 图片来自美团点评技术博客

![](http://tech.meituan.com/img/java-hashmap/hashMap%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE.png)


#### 一些变量
```java
    //数组，第一次使用时必须被初始化，必要时进行扩容，被分配之后长度一直会是 2 的幂，可以在一些场合容忍长度为 0
    transient Node<K,V>[] table; 
    //用来探测结构性修改的，实现 fast-fail 机制
    transient int modCount; 
    //下次 resize 的阈值，值为 capacity * load factor
    int threshold;
```

#### 构造器
HashMap 总共提供了`四种`不同形式的构造器，让我们一起来看看分别有什么特点。

##### 第一种
```java
    public HashMap(int initialCapacity, float loadFactor) { //带初始化容量和负载因子的
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY) //限定最大容量
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

##### 第二种
```java
    public HashMap(int initialCapacity) { //只带有初始化容量，负载因子使用缺省值，即 0.75f
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

##### 第三种
```java
    public HashMap() { //没有初始化容量，只有缺省值的负载因子
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

##### 第四种
```java
    public HashMap(Map<? extends K, ? extends V> m) { //传一个 Map 进去
        this.loadFactor = DEFAULT_LOAD_FACTOR; //使用负载因子的缺省值
        putMapEntries(m, false); //调用 putMapEntries 方法将 Map 中的数值迁移到 HashMap 中，下面看看这个方法的实现
    }
```

#### putMapEntries

`evict` 这个参数是 `false` 的时候表示是在创建 `HashMap` 时调用的这个函数，反之则是在创建之后调用的

```java
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) { 
            if (table == null) { // 如果此时数组尚未初始化
                float ft = ((float)s / loadFactor) + 1.0F; //计算实际所需容量 + 1
                int t = ((ft < (float)MAXIMUM_CAPACITY) ? //判断容量是否溢出
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    //如果大于原先设定的阈值，则通过调用 tableSizeFor 方法寻找最接近满足条件的 2 的幂
                    threshold = tableSizeFor(t); 
            }
            else if (s > threshold) //如果数组已经初始化完成，判断是否超过 resize 的阈值，若是扩容
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict); //拿到 key，value 之后插入哈希表
            }
        }
    }
```

#### get & getNode
```java
    public V get(Object key) { //根据 key 获取值的方法
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    //final 修饰，不可变的方法
    final Node<K,V> getNode(int hash, Object key) { 
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k; //参数很多，慢慢看
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) { //hash & (length-1)得到对象的保存位  
            if (first.hash == hash && // 总是先检查首节点的 hash 值，再接着比对 key 的具体内容
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first; //符合则返回首节点
            if ((e = first.next) != null) { //如果首节点仅是 hash 值命中，key 的内容不一样的话，说明真正要找的值在后挂链表或者后挂红黑树里面
                if (first instanceof TreeNode) //判断是否是树节点
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key); //是的话调用红黑树的函数获取值
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) 
                        return e;
                } while ((e = e.next) != null); //否则的话，在链表中一次寻找
            }
        }
        return null;
    }
```

#### put & putVal
```java
    public V put(K key, V value) { //放入 key value 的方法
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0) //如果数组为初始化过或者 lenght = 0
            n = (tab = resize()).length; //调用 resize 方法初始化并返回 length
        if ((p = tab[i = (n - 1) & hash]) == null) //如果数组中还没有这个 bucket，直接新增一个数组项
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k)))) //否则先找到 bucket 的位置
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); //如果后挂红黑树，则需要调用红黑树的方法插入节点
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) { //e为空，表示已到表尾也没有找到key值相同节点，则新建节点  
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash); //转化为红黑树解决冲突
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) //容许 key 为 null  
                        break;
                    p = e; //p指向下一个节点  
                }
            }
            if (e != null) { // existing mapping for key，已经存在该元素
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize(); //扩容
        afterNodeInsertion(evict);
        return null;
    }
```

#### remove & removeNode
```java
    public V remove(Object key) { // 删除节点的方法
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ? //调用 removeNode 方法
            null : e.value;
    }

    /** @param value 允许传个值进去，用于比较删除节点时 value 是否相等
     *  @param matchValue true 意味着当且仅当 value 也相等时删除节点
     *  @param movable if false do not move other nodes while removing
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) { //参数略多啊...慢慢看
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) { //如果这个位置有首节点的话
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k)))) //找到了这个 key
                node = p;
            else if ((e = p.next) != null) { //否则在链表或者红黑树中寻找
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

#### resize

核心方法 `resize`，既可以用于初始化 table 数组，又可以完成`扩容两倍`的本职工作。下面的分析摘自美团点评技术博客：

> 下面我们讲解下 JDK 1.8 做了哪些优化。经过观测可以发现，我们使用的是 2 次幂的扩展(指长度扩为原来 2 倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动 2 次幂的位置。看下图可以明白这句话的意思，n 为 table 的长度，图（a）表示扩容前的 key1 和 key2 两种 key 确定索引位置的示例，图（b）表示扩容后 key1 和 key2 两种 key 确定索引位置的示例，其中 hash1 是 key1 对应的哈希与高位运算结果。

![](http://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE1.png)

> 元素在重新计算 hash 之后，因为 n 变为 2 倍，那么 n-1 的 mask 范围在高位多 1 bit (红色)，因此新的 index 就会发生这样的变化：

![](http://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE2.png)

> 因此，我们在扩充 HashMap 的时候，不需要像 JDK 1.7 的实现那样重新计算 hash，只需要看看原来的 hash 值新增的那个 bit 是 1 还是 0 就好了，是 0 的话索引没变，是 1 的话索引变成 “原索引 + oldCap”，可以看看下图为 16 扩充为 32 的 resize 示意图：

![](http://tech.meituan.com/img/java-hashmap/jdk1.8%20hashMap%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {                       //如果容量大于 0，说明已经有元素存在里面了
            if (oldCap >= MAXIMUM_CAPACITY) {   //若超过 1>>30 大小，无法扩容只能改变阈值
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 不溢出的情况下，容量阈值均扩为 2 倍，最小为 16
        }
        else if (oldThr > 0) // 如果 oldCap <= 0，初始容量为阈值 threshold  
            newCap = oldThr;
        else {               //还记得在 putVal 中调用 resize 可以完成初始化吗？就是这里，使用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) { //计算新的扩容上限
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr; //更新阈值
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //创建一个初始容量为新 hash 表长度的newTab数组
        table = newTab;
        if (oldTab != null) {                                   //如果旧表不为空，则按顺序将旧表中的元素重定向到新表，把每个bucket 都移动到新的 buckets 中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;                                    //e 按序指向每个链表中的头结点
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)                         //如果仅有头结点
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); //红黑树分裂节点
                    else {                                      //保持原有的顺序，链表优化重 hash 的代码块
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {       //原索引：如果(e.hash & oldCap)==true，表明(e.hash & (newCap - 1))还会和 e.hash & (oldCap - 1)一样。因为 oldCap 和 newCap 是 2 的幂，并且 newCap是oldCa一个二进制的1向高位移动了一位(e.hash & oldCap) == 0就代表了(e.hash & (newCap - 1))还会 和 e.hash & (oldCap - 1)一样。
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {                              //原索引 + oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {                   //原索引放到 bucket 里
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {                   //原索引 + oldCap 放到 bucket 里
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

#### HashMap 中的红黑树结构

这部分貌似别的博客都很少涉及，那我来讲一讲吧！

##### TreeNode

树节点的类中主要有以下属性：父节点、左节点、右节点、前置节点、颜色值，继承自 `LinkedHashMap.Entry<K,V>`

```java
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
    }

    //这里还有前指针，后指针，哈希值，数值 value 等一些属性
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

```

##### treeifyBin

我们在 `putVal` 方法中谈到，当节点数已经超过 `TREEIFY_THRESHOLD` 之后，需要调用 `treeifyBin` 方法将链表转化为红黑树

```java
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) //当哈希表为空或者长度小于进行树形化的阈值 64 的话，就先扩容
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null; //定义红黑树的头尾节点
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null); //新建红黑树节点，值为 e，建立前后节点的连接关系
                if (tl == null) 
                    hd = p;     //头结点
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null) //让数组 tab 的第一个元素指向红黑树的头结点
                hd.treeify(tab); //传入 tab 开始树形化
        }
    }
```

##### treeify

```java
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (root == null) { //指定树的根节点
                x.parent = null;
                x.red = false; //黑色
                root = x;
            }
            else { //有了根节点后，进入内部循坏，用双层循坏来比较其他节点的 hash 值跟当前节点的 hash 值，然后确定左右子节点
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) { //内部循坏开始
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h) 
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                                (kc = comparableClassFor(k)) == null) ||
                                (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);

                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }
```

##### untreeify

```java
    //解除树形化，返回一个链表
    final Node<K,V> untreeify(HashMap<K,V> map) {
        Node<K,V> hd = null, tl = null;
        for (Node<K,V> q = this; q != null; q = q.next) {
            Node<K,V> p = map.replacementNode(q, null);
            if (tl == null)
                hd = p; //找到头结点
            else
                tl.next = p; //一次相连
            tl = p;
        }
        return hd;
    }
```

##### find & getTreeNode

```java
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        TreeNode<K,V> p = this;
        do {
            int ph, dir; K pk;
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            else if ((pk = p.key) == k || (k != null && k.equals(pk))) //命中
                return p;
            else if (pl == null) //左子树为空
                p = pr;
            else if (pr == null) //右子树为空
                p = pl;
            else if ((kc != null ||
                        (kc = comparableClassFor(k)) != null) &&
                        (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.find(h, k, kc)) != null) //递归查询
                return q;
            else
                p = pl;
        } while (p != null);
        return null;
    }

    /**
        * Calls find for root node.
        */
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);  //从根节点开始调用 find 方法
    }
```

##### putTreeVal

```java
    final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        TreeNode<K,V> root = (parent != null) ? root() : this; //获取根节点
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (k != null && k.equals(pk))) //已存在直接返回
                return p;
            else if ((kc == null &&
                        (kc = comparableClassFor(k)) == null) ||
                        (dir = compareComparables(kc, k, pk)) == 0) {
                if (!searched) { //如果当前节点和要添加的节点哈希值相等，但是两个节点的键不是一个类，挨个对比左右孩子
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null &&
                            (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                            (q = ch.find(h, k, kc)) != null))
                        return q; //如果找到了，说明已存在
                }
                dir = tieBreakOrder(k, pk); //哈希值相等但是 key 无法比较时的方法
            }

            //要插入的节点比当前节点小就插到左子树，大就插到右子树
            TreeNode<K,V> xp = p;
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                Node<K,V> xpn = xp.next;
                TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                xp.next = x;
                x.parent = x.prev = xp;
                if (xpn != null)
                    ((TreeNode<K,V>)xpn).prev = x;
                moveRootToFront(tab, balanceInsertion(root, x)); //红黑树是一种平衡树，插入节点后需要进行平衡性调整，左旋右旋之类的
                return null;
            }
        }
    }
```

### Conclusion

HashMap 是根据键的 hashCode 值存储数据，可以通过 key 直接定位到它的 value，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap 最多只允许一条记录的 key 为 null，允许多条记录的 value 为 null。HashMap 是非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。HashMap 解决冲突的方法是链表法，节点数过多或者过少时，可以在链表与红黑树之间进行转换。在 tab 容量达到阈值时可以进行 resize，一般情况下变为原先的 2 倍，扩容时线程并发可能导致链表无限循环 Bug。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>