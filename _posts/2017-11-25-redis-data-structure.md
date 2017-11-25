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
```C
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
```C
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
```C
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
```C
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
```C
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

### 未完待续···
 
### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang[AT]foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>



