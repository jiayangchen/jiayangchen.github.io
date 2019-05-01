---
layout: post
title: "MySQL InnoDB 中的 redo/undo log"
subtitle: "讲到 InnoDB、MVCC 等概念时，我们时常听到 redo log 和 undo log 的名字，那么二者的作用是什么呢"
date: 2019-04-13
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - MySQL
---

### 写在前面

讲到 `InnoDB`、`MVCC` 等概念时，我们时常听到 `redo log` 和 `undo log` 的名字，那么二者的作用是什么呢？其实二者并非事务操作独有，索引更新时也会记录 `redo/undo log`，甚至记录 `undo log` 时也会记录 `redo log`，而本文聚焦于事务方面的 `redo/undo log`。

#### 什么是 redo log

`MySQL` 中使用了大量内存 `Cache` 区域，对数据的修改操作会先直接修改内存中的 `Page`，但这些页不会立刻同步磁盘，这时内存中的数据已经和磁盘上的不一致了，我们称这种 `Page` 为脏页。试想一下这时候如果数据库宕机了，内存中这部分被修改的数据记录就丢失了，重启后也没办法恢复。

因此为了保证数据的安全性，在修改内存中的 `Page` 之后 `InnoDB` 会写 `redo log`，因为 `redo log` 是顺序写入的，而众所周知磁盘的顺序读写的速度远大于随机读写，因此这部分日志写操作对性能影响较小。然后，`InnoDB` 会在事务提交前将 `redo log` 保存到磁盘中。这里所说的 `redo log` 是物理日志而非逻辑日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页（恢复数据页，且只能恢复到最后一次提交的位置）。

当数据库意外重启时，会根据 `redo log` 进行数据恢复，如果 `redo log` 中有事务提交，则进行事务提交修改数据。

#### 什么是 undo log

与 `redo log` 不同，`undo log` 一般是逻辑日志，根据每行记录进行记录。例如当 `DELETE` 一条记录时，`undo log` 中会记录一条对应的 `INSERT` 记录，反之亦然当 `UPDTAE` 一条记录时，它记录一条对应反向 `UPDATE` 记录。

> 分布式事务 Seata 的 GTS 模式就是这么实现的，只不过这个过程放在了客户端代码里，而且支持的 SQL 类型有要求

当数据被修改时除了会记录 `redo log` 还会记录 `undo log`，通过 `undo log` 一方面可以实现事务回滚，另一方面可以根据 `undo log` 回溯到某个特定的版本的数据，实现 `MVCC` 的功能。

### 为什么要引入 redo log ？

> 假设现在只有 undo log 机制。

事务提交前为了保证突发宕机不丢更新，肯定要将 `undo log` 刷到磁盘，但这会造成多次磁盘 `IO`，但是考虑到日志文件是顺序写入，性能还好。事务提交后，需要将内存中的脏页立即同步到磁盘的数据文件中，这会造成至少一次磁盘随机 `IO`，影响性能。

因此数据库设计者就思考，如果事务提交后，脏页数据能在内存中缓存一段时间，而不需要立即被同步到磁盘文件，那么就能合并多次随机 `IO` 来提高性能。但是这样就会丧失事务的持久性，因为假如事务已经提交，数据却还在内存中时宕机了，重启后因为只有 `undo log` 无法恢复被提交的数据。

因此设计者们引入了另外一种机制来实现持久化，即 `redo log`。在事务提交前，只要将 `redo log` 持久化即可，不需要将数据持久化。当系统宕机时，虽然数据还没来得及刷回磁盘，但是 `redo log` 已经持久化了，系统可以根据 `redo log` 的内容，将所有数据恢复到最新的状态。

#### 只靠 undo log 行么？

只有 `undo log` 的情况下就必须保证事务提交前，脏页必须刷回磁盘，否则宕机时这部分数据修改就在内存中丢失了，破坏了持久性，但是如上文提到的，这种办法性能太差，必须改进。

#### 只靠 redo log 行么？

只有 `redo log` 的情况下就不能在事务提交前刷脏，我们假设有个大事务更新数据量很大，更新完一部分必须刷一部分脏页回磁盘然后再 `load` 其他的数据进内存操作，此时如果更新到一半宕机了，那么数据库里存在一部分新数据一部分旧数据，而事务又没有提交，因此应该回滚，只有 `redo log` 的情况下无法完成。

### redo/undo log 的持久化

`redo log` 由两部分组成，一部分是内存中的 `redo log buffer`，这部分是易失的，重启就没了；二是磁盘上的 `redo log file`，是持久化的。

`InnoDB` 通过 `force log at commit` 技术来实现事务的持久化特性。为了保证每次 `redo log` 都能写入磁盘上的日志文件中，每次将内存中的 `redo log buffer` 内容同步磁盘时都会调用一次 `fsync`。

> OS 中 write 和 fsync 是不同的操作，我们以为调用了 write 就万事大吉，数据一定到磁盘了，其实不一定，通常情况下 write 只是到了磁盘 IO 缓冲区，何时 fsync 由 OS 控制，这里通过程序强制调用来保证日志一定刷到磁盘

![](http://ww1.sinaimg.cn/large/c3beb895ly1g212vf2ljjj20u20reju7.jpg)

### 参考资料

- [redo/undo log、binlog 的详解及其区别](https://www.jianshu.com/p/57c510f4ec28)
- [详细分析MySQL事务日志(redo log和undo log)](https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html)