---
layout: post
title: "InnoDB MVCC 原理及实现"
subtitle: ""
date: 2019-04-13
author: "ChenJY"
header-img: "img/mysql.jpg"
catalog: true
tags: 
    - MySQL
---

### 什么是 MVCC

`MVCC (Multiversion Concurrency Control)` 中文全程叫**多版本并发控制**，是现代数据库（包括 `MySQL`、`Oracle`、`PostgreSQL` 等）引擎实现中常用的处理读写冲突的手段，**目的在于提高数据库高并发场景下的吞吐性能**。

如此一来，不同事务并发过程中，`SELECT` 操作可以不加锁而是通过 `MVCC` 机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别，从而解决并发场景下的读写冲突。

### InnoDB 中的 MVCC

本文聚焦于 `MySQL` 中的 `MVCC` 实现，从 `《高性能 MySQL》`一书中对 `MVCC` 的介绍可知：

1. `MySQL` 中 `InnoDB` 引擎支持 `MVCC`
2. 应对高并发事务, `MVCC` 比单纯的加行锁更有效, 开销更小
3. `MVCC` 在读已提交`（Read Committed）`和可重复读`（Repeatable Read）`隔离级别下起作用
4. `MVCC` 既可以基于**乐观锁**又可以基于**悲观锁**来实现

### Redo Log、Undo Log

关于 InnoDB 我们时常听到 Redo Log 和 Undo Log 的名字，那么二者的作用是什么呢？其实二者并非事务操作独有，索引更新时也会记录 Redo/Undo Log，而本文聚焦于事务方面的 Redo/Undo Log。

#### Redo Log 概念

MySQL 中使用了大量内存 Cache 区域，对数据的修改操作会先直接修改内存中的 Page，但这些页不会立刻同步磁盘，这时内存中的数据已经和磁盘上的不一致了，我们称这种 Page 为脏页。试想一下这时候如果数据库宕机了，内存中这部分被修改的数据记录就丢失了，重启后也没办法恢复。

因此为了保证数据的安全性，在修改内存中的 Page 之后 InnoDB 会写 Redo Log，因为 Redo Log 是顺序写入的，而众所周知磁盘的顺序读写的速度远大于随机读写，因此这部分日志写操作对性能影响较小。然后，InnoDB 会在事务提交前将 Redo Log 保存到磁盘中。这里所说的 Redo Log 是物理日志而非逻辑日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页（恢复数据页，且只能恢复到最后一次提交的位置）。

当数据库意外重启时，会根据 Redo Log 进行数据恢复，如果 Redo Log 中有事务提交，则进行事务提交修改数据。

#### Undo Log 概念

与 Redo Log 不同，Undo Log 一般是逻辑日志，根据每行记录进行记录。例如当 DELETE 一条记录时，Undo Log 中会记录一条对应的 INSERT 记录，反之亦然当 UPDTAE 一条记录时，它记录一条对应反向 UPDATE 记录。

> 分布式事务 Seata 的 GTS 模式就是这么实现的，只不过这个过程放在了客户端代码里，而且支持的 SQL 类型有要求

当数据被修改时除了会记录 Redo Log 还会记录 Undo Log，通过 Undo Log 一方面可以实现事务回滚，另一方面可以根据 Undo Log 回溯到某个特定的版本的数据，实现 MVCC 的功能。

#### Redo/Undo Log 的持久化

Redo/Undo Log 由两部分组成，一部分是内存中的 Redo/Undo Log Buffer，这部分是易失的，重启就没了；二是磁盘上的 Redo/Undo Log File，是持久化的。

InnoDB 通过 force log at commit 技术来实现事务的持久化特性。为了保证每次 Redo/Undo Log 都能写入磁盘上的日志文件中，每次将内存中的 Redo/Undo Log Buffer 内容同步磁盘时都会调用一次 fsync。

> OS 中 write 和 fsync 是不同的操作，我们以为调用了 write 就万事大吉，数据一定到磁盘了，其实不一定，通常情况下 write 只是到了磁盘 IO 缓冲区，何时 fsync 由 OS 控制，这里通过程序强制调用来保证日志一定刷到磁盘

![](http://ww1.sinaimg.cn/large/c3beb895ly1g212vf2ljjj20u20reju7.jpg)

### InnoDB MVCC 实现原理

`InnoDB` 中 `MVCC` 的实现方式为：每一行记录都有两个隐藏列：`DATA_TRX_ID`、`DATA_ROLL_PTR`（如果没有主键，则还会多一个隐藏的主键列）。

![](http://ww1.sinaimg.cn/large/c3beb895ly1g211pquwtyj20hu03eaa3.jpg)

#### DATA_TRX_ID 

记录最近更新这条行记录的`事务 ID`，大小为 `6` 个字节

#### DATA_ROLL_PTR

表示指向该行回滚段`（rollback segment）`的指针，大小为 `7` 个字节，`InnoDB` 便是通过这个指针找到之前版本的数据。该行记录上所有旧版本，在 `undo` 中都通过链表的形式组织。

### Undo Log 链

我们用一个例子来看一下 DATA_TRX_ID 和 DATA_ROLL_PTR 两个参数是如何起作用的。



### ReadView

在 Read Uncommitted 隔离级别下，直接读取版本的最新记录就 OK，对于 SERIALIZABLE 隔离级别，则是通过加锁互斥来访问数据，因此不需要 MVCC 的帮助。因此 MVCC 运行在 Read Committed 和 Repeatable Read 这两个隔离级别下，当 InnoDB 隔离级别设置为二者其一时，在 SELECT 数据时就会用到版本链

> 核心问题是版本链中哪些版本对当前事务可见？

InnoDB 为了解决这个问题，设计了 ReadView（可读视图）的概念，








