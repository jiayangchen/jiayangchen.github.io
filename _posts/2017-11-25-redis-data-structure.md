---
layout: post
title: "Redis 内部数据结构及存储方式"
subtitle: "关于 Redis 内部的五种存储数据结构的底层实现"
date: 2017-11-25
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - redis 设计与实现
    - 读书笔记
---

我们都知道，Redis 是一个 KV 数据库，常用于缓存，现在也承担一部分数据持久化的工作。而其中，Redis 内部的每个键值对 `<K,V>` 都是由对象组成的，其中键总是一个字符串对象，而值可以是 String、List、Dictionary、Set 和 SortedSet 这五种，多样的存储对象也给 Redis 提供了更为丰富的使用场景，经由开发者的定制化程度高，一定程度上促使了 Redis 使用的快速普及。本文中，主要讨论这五种数据结构的底层实现，我们会发现，其实这五种结构都是彼此有关联的，并不是各个独立设计产生，Redis 的设计者很巧妙地将一些共性的特征通过合理的组织使之能更好的被复用。

### 简单动态字符串 SDS
#### SDS 简介
Redis 虽然是基于 C 语言开发的，但是及时是 String 这样简单的数据结构，Redis 内部也不是直接使用了 C 的封装，而是自己封装了一层抽象类型，命名为简单动态字符串（Simple Dynamic String。仅当无需对字符串值进行修改的场景下，例如日志打印，Redis 才会直接使用 C 字符串，而 Redis 中，包含字符串值的键值对都是通过 SDS 实现的。

#### SDS 类
```C++
struct sdshdr {
    // buf 已占用长度
    int len;
    // buf 剩余可用长度
    int free;
    // 实际保存字符串数据的地方
    char buf[];
};
```
需要注意的是，Redis 保留了 C 语言中字符串以 `\0` 结尾的传统，但是计算长度时不包含在内，这使得 SDS 可以直接使用一部分 C 语言的函数来完成功能。
#### SDS 设计的优点

1. 可以以常数级别的复杂度得到字符串的长度。而不是像 C 语言那样以扫描一遍计数的方式，复杂度达到了 O(N)，这点和 Java 中数组对象存储 length 属性有异曲同工之妙。
2. 杜绝缓冲区溢出。因为记录了长度信息，当操作需要对字符串内容进行改变时，会先检查是否有足够的长度进行扩容。
3. 更好的扩容策略。这里就要提到 Redis 内部的扩容策略了，我们对于字符串的操作经常会引起扩容或者是缩容释放内存，Redis 又是一个内存数据库，其中值的修改次数非常频繁，因此每次只扩容所需大小会造成频繁扩容，影响性能。因此 Redis 采用了 `空间预分配` 和 `惰性空间释放` 两种优化策略:

##### 空间预分配
实际上就是每次多分配一些空余的，之后不用每次都进行扩容。策略是，如果 SDS 进行修改之后，len 属性值小于 1MB，程序分配和 len 属性同样大小的空闲空间；否则的话，分配 1MB 空闲空间。

##### 惰性空间释放
缩容时，简单来说就是不直接释放空间，而是在 free 属性中记录，等着下次万一有用的时候直接用。

### 链表
#### Redis 内部链表的数据结构
Redis 内部实现的是双向链表,先来看看链表节点的结构体代码：
```C++
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```
接下去时链表类的结构体：
```C++
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;
```
于是乎，一个相对完整的链表结构图，如下所示：
![](http://redisbook.com/_images/graphviz-5f4d8b6177061ac52d0ae05ef357fceb52e9cb90.png)

### 字典
又称为符号表或者关系数组，是一种保存键值对的抽象数据结构。字典中每个键都是唯一的，可以根据键来删改查对应的值，Redis 的数据库就是使用字典实现的。
#### 哈希表
字典的底层实现是哈希表，如下：
```C++
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```
table 属性是一个数组， 数组中的每个元素都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对。size 属性记录了哈希表的大小， 也即是 table 数组的大小， 而 used 属性则记录了哈希表目前已有节点（键值对）的数量。sizemask 属性的值总是等于 size - 1 ， 这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面。
![](http://redisbook.com/_images/graphviz-bd3eecd927a4d8fc33b4a1c7f5957c52d67c5021.png)

#### 哈希表节点
```C++
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```
![](http://redisbook.com/_images/graphviz-d2641d962325fd58bf15d9fffb4208f70251a999.png)
#### 字典的数据结构
```C++
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```
`type` 属性和 `privdata` 属性是针对不同类型的键值对， 为创建多态字典而设置的。`ht` 属性是一个包含两个项的数组， 数组中的每个项都是一个 `dictht` 哈希表， 一般情况下， 字典只使用 `ht[0]` 哈希表， `ht[1]` 哈希表只会在对 `ht[0]` 哈希表进行 rehash 时使用。除了 `ht[1]` 之外， 另一个和 rehash 有关的属性就是 `rehashidx` ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。
![](http://redisbook.com/_images/graphviz-e73003b166b90094c8c4b7abbc8d59f691f91e27.png)
#### 哈希算法
哈希算法部分不多说了，熟悉 Java HashMap 的同学应该不会陌生，其实大同小异，解决哈希冲突的方法也是采用 `链表`。
#### 重点之 Rehash
当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。
当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：
* 服务器目前没有在执行 `BGSAVE` 命令或者 `BGREWRITEAOF` 命令， 并且哈希表的负载因子大于等于 1 ；
* 服务器目前正在执行 `BGSAVE` 命令或者 `BGREWRITEAOF` 命令， 并且哈希表的负载因子大于等于 5 ；
之所以根据 BGSAVE 命令或 BGREWRITEAOF 命令是否正在执行来划分服务器执行扩展操作所需的负载因子， 这是因为在执行 BGSAVE 命令或 BGREWRITEAOF 命令的过程中， Redis 需要创建当前服务器进程的子进程， 而大多数操作系统都采用写时复制`（copy-on-write）`技术来优化子进程的使用效率， 所以在子进程存在期间， 服务器会提高执行扩展操作所需的负载因子， 从而尽可能地避免在子进程存在期间进行哈希表扩展操作， 这可以避免不必要的内存写入操作， 最大限度地节约内存。

具体 `Rehash` 方法也很简单，主要分为三步走：
1. 为字典的 `ht[1]` 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 `ht[0]` 当前包含的键值对数量 （也即是 ht[0].used 属性的值）：
* 如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；
* 如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n 。
2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上。
3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备。

#### 渐进式哈希
扩展或收缩哈希表需要将 ht[0] 里面的所有键值对 rehash 到 ht[1] 里面， 但是， 这个 rehash 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。原因在于如果 Redis ht[0] 中保存的键值对数目过多，那么一次性哈希的方式可能导致服务直接不可用。为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1] ， 而是分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1] 。

哈希表渐进式 `rehash` 的详细步骤：
1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。
4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。

>因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

### 跳跃表
我觉得比较复杂，看得挺晕的，不如专门讲《数据结构与算法分析》那本书里讲的详细，可能是涉及到了具体代码层面实现的问题，因为图画的不太好懂，这部分建议大家去看看专门讲跳表的书，会对跳表的概念理解得更好，并不是很复杂的数据结构。

### 整数集合
```C++
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```
#### 整数集合的升级
每当我们要将一个新元素添加到整数集合里面， 并且新元素的类型比整数集合现有所有元素的类型都要长时， 整数集合需要先进行升级， 然后才能将新元素添加到整数集合里面。升级步骤也很简单：

1. 根据新插入的元素的类型，来判断是否需要升级操作，如果需要，则开始扩充底层数组大小
2. 将原本储存的元素都转换成新的类型之大小，然后移放到正确的位置上
3. 将新元素放在空缺的位置上

> 因为每次向整数集合添加新元素都可能会引起升级， 而每次升级都需要对底层数组中已有的所有元素进行类型转换， 所以向整数集合添加新元素的时间复杂度为 `O(N)` 。

### 压缩列表
先看一个压缩列表的事例
![](http://redisbook.com/_images/graphviz-071fe5086440a360087904af9a5f78e8e02c2d8d.png)

* 列表 `zlbytes` 属性的值为 0x50 （十进制 80）， 表示压缩列表的总长为 80 字节。
* 列表 `zltail` 属性的值为 0x3c （十进制 60）， 这表示如果我们有一个指向压缩列表起始地址的指针 p ， 那么只要用指针 p 加上偏移量 60 ， 就可以计算出表尾节点 entry3 的地址。
* 列表 `zllen` 属性的值为 0x3 （十进制 3）， 表示压缩列表包含三个节点。

#### 压缩列表节点
每个压缩列表节点都由 `previous_entry_length`、`encoding`、`content` 三个部分组成：
![](http://redisbook.com/_images/graphviz-cc6b40e182bfc142c12ac0518819a2d949eafa4a.png)

* 节点的 `previous_entry_length` 属性以字节为单位，顾名思义，记录了压缩列表中前一个节点的长度。因为节点的 previous_entry_length 属性记录了前一个节点的长度， 所以程序可以通过指针运算， 根据当前节点的起始地址来计算出前一个节点的起始地址。
* 节点的 `encoding` 属性记录了节点的 `content` 属性所保存数据的类型以及长度
* 节点的 `content` 属性负责保存节点的值， 节点值可以是一个字节数组或者整数， 值的类型和长度由节点的 `encoding` 属性决定。

#### 重点之连锁更新
简而言之，就是因为新增节点之后原本保存前驱节点长度的 `previous_entry_length` 属性空间不够，需要重新扩大时，会导致它的后继节点也需要扩大其 `previous_entry_length` 属性空间大小，因为他们彼此都是保存的前者长度的。

> 因为连锁更新在最坏情况下需要对压缩列表执行 N 次空间重分配操作， 而每次空间重分配的最坏复杂度为 O(N) ， 所以连锁更新的最坏复杂度为 O(N^2)。

### 对象
Redis 中的每个对象都由一个 redisObject 结构表示：
```C++
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```
对于 Redis 数据库保存的键值对来说， 键总是一个字符串对象， 而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种。

#### 字符串对象
##### 编码方式
字符串对象的编码可以是 `int` 、 `raw` 或者 `embstr` 。

* 如果一个字符串对象保存的是整数值， 并且这个整数值可以用 long 类型来表示， 那么字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面（将 void* 转换成 long ）， 并将字符串对象的编码设置为 int 。
* 如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度大于 39 字节， 那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值， 并将对象的编码设置为 `raw` 。
* 如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度小于等于 39 字节， 那么字符串对象将使用 `embstr` 编码的方式来保存这个字符串值。

**PS：**
`embstr` 编码的字符串对象在执行命令时， 产生的效果和 `raw` 编码的字符串对象执行命令时产生的效果是相同的， 但使用 `embstr` 编码的字符串对象来保存短字符串值有以下好处：

1. `embstr` 编码将创建字符串对象所需的内存分配次数从 `raw` 编码的两次降低为一次。
2. 释放 `embstr` 编码的字符串对象只需要调用一次内存释放函数， 而释放 raw 编码的字符串对象需要调用两次内存释放函数。
3. 因为 `embstr` 编码的字符串对象的所有数据都保存在一块连续的内存里面， 所以这种编码的字符串对象比起 `raw` 编码的字符串对象能够更好地利用缓存带来的优势。

##### 编码转换
对于 `int` 编码的字符串对象来说， 如果我们向对象执行了一些命令， 使得这个对象保存的不再是整数值， 而是一个字符串值， 那么字符串对象的编码将从 `int` 变为 `raw` 。

#### 列表对象
列表对象的编码可以是 `ziplist` 或者 `linkedlist` 。`ziplist` 编码的列表对象使用压缩列表作为底层实现， 每个压缩列表节点（entry）保存了一个列表元素。另一方面， `linkedlist` 编码的列表对象使用双端链表作为底层实现， 每个双端链表节点（node）都保存了一个字符串对象， 而每个字符串对象都保存了一个列表元素。

##### 编码转换
当列表对象可以同时满足以下两个条件时， 列表对象使用 `ziplist` 编码：

1. 列表对象保存的所有字符串元素的长度都小于 64 字节；
2. 列表对象保存的元素数量小于 512 个；

不能满足这两个条件的列表对象需要使用 `linkedlist` 编码。

#### 哈希对象
哈希对象的编码可以是 `ziplist` 或者 `hashtable` 。

#### 集合对象
集合对象的编码可以是 `intset` 或者 `hashtable` 。
有序集合的编码可以是 `ziplist` 或者 `skiplist` 。

### 内存回收
因为 C 语言并不具备自动的内存回收功能， 所以 Redis 在自己的对象系统中构建了一个引用计数（reference counting）技术实现的内存回收机制， 通过这一机制， 程序可以通过跟踪对象的引用计数信息， 在适当的时候自动释放对象并进行内存回收。

```C++
typedef struct redisObject {
    // ...
    // 引用计数
    int refcount;
    // ...
} robj;
```

### 参考资料
* <a href="http://product.dangdang.com/23501734.html" target="_blank"><b>《Redis 设计与实现》</b></a> 黄建宏 著

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang[AT]foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>



