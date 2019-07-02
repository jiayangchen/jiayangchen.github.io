---
layout: post
title: "MySQL InnoDB MVCC 机制的原理及实现"
subtitle: "多版本并发控制，是现代数据库引擎实现中常用的处理读写冲突的手段，目的在于提高数据库高并发场景下的吞吐性能。"
date: 2019-06-22
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - MySQL
---

- [什么是 MVCC](#什么是-mvcc)
- [为什么需要 MVCC](#为什么需要-mvcc)
- [InnoDB 中的 MVCC](#innodb-中的-mvcc)
- [InnoDB MVCC 实现原理](#innodb-mvcc-实现原理)    
    - [DATA_TRX_ID](#data_trx_id)    
    - [DATA_ROLL_PTR](#data_roll_ptr)    
    - [DB_ROW_ID](#db_row_id)
- [如何组织 Undo Log 链](#如何组织-undo-log-链)
- [如何实现一致性读 —— ReadView](#如何实现一致性读--readview)    
    - [RR 下的 ReadView 生成](#rr-下的-readview-生成)    
    - [RC 下的 ReadView 生成](#rc-下的-readview-生成)    
- [举个例子](#举个例子)    
    - [RC 下的 MVCC 判断流程](#rc-下的-mvcc-判断流程)    
    - [RR 下的 MVCC 判断流程](#rr-下的-mvcc-判断流程)    
- [一个争论点](#一个争论点)
- [总结](#总结)
- [参考资料](#参考资料)

### 什么是 MVCC

`MVCC (Multiversion Concurrency Control)` 中文全程叫**多版本并发控制**，是现代数据库（包括 `MySQL`、`Oracle`、`PostgreSQL` 等）引擎实现中常用的处理读写冲突的手段，**目的在于提高数据库高并发场景下的吞吐性能**。

如此一来不同的事务在并发过程中，`SELECT` 操作可以不加锁而是通过 `MVCC` 机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别，从而解决并发场景下的读写冲突。

下面举一个多版本读的例子，例如两个事务 `A` 和 `B` 按照如下顺序进行更新和读取操作

![](http://ww1.sinaimg.cn/large/c3beb895gy1g2lrq1k5r0j20iy0hkt9l.jpg)

在事务 `A` 提交前后，事务 `B` 读取到的 `x` 的值是什么呢？答案是：事务 `B` 在不同的隔离级别下，读取到的值不一样。

1. 如果事务 `B` 的隔离级别是读未提交（RU），那么两次读取均读取到 `x` 的最新值，即 `20`。
2. 如果事务 `B` 的隔离级别是读已提交（RC），那么第一次读取到旧值 `10`，第二次因为事务 `A` 已经提交，则读取到新值 20。
3. 如果事务 `B` 的隔离级别是可重复读或者串行（RR，S），则两次均读到旧值 `10`，不论事务 `A` 是否已经提交。

可见在不同的隔离级别下，数据库通过 `MVCC` 和隔离级别，让事务之间并行操作遵循了某种规则，来保证单个事务内前后数据的一致性。

### 为什么需要 MVCC

`InnoDB` 相比 `MyISAM` 有两大特点，一是支持事务而是支持行级锁，事务的引入带来了一些新的挑战。相对于串行处理来说，并发事务处理能大大增加数据库资源的利用率，提高数据库系统的事务吞吐量，从而可以支持可以支持更多的用户。但并发事务处理也会带来一些问题，主要包括以下几种情况：

1. 更新丢失（`Lost Update`）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题 —— 最后的更新覆盖了其他事务所做的更新。如何避免这个问题呢，最好在一个事务对数据进行更改但还未提交时，其他事务不能访问修改同一个数据。

2. 脏读（`Dirty Reads`）：一个事务正在对一条记录做修改，在这个事务并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些尚未提交的脏数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做 **“脏读”**。

3. 不可重复读（`Non-Repeatable Reads`）：一个事务在读取某些数据已经发生了改变、或某些记录已经被删除了！这种现象叫做“不可重复读”。

4. 幻读（`Phantom Reads`）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为 **“幻读”**。

以上是并发事务过程中会存在的问题，解决更新丢失可以交给应用，但是后三者需要数据库提供事务间的隔离机制来解决。实现隔离机制的方法主要有两种：

1. 加读写锁

2. 一致性快照读，即 `MVCC`

但本质上，隔离级别是一种在并发性能和并发产生的副作用间的妥协，通常数据库均倾向于采用 `Weak Isolation`。

### InnoDB 中的 MVCC

本文聚焦于 `MySQL` 中的 `MVCC` 实现，从 `《高性能 MySQL》`一书中对 `MVCC` 的介绍可知：

1. `MySQL` 中 `InnoDB` 引擎支持 `MVCC`
2. 应对高并发事务, `MVCC` 比单纯的加行锁更有效, 开销更小
3. `MVCC` 在读已提交`（Read Committed）`和可重复读`（Repeatable Read）`隔离级别下起作用
4. `MVCC` 既可以基于**乐观锁**又可以基于**悲观锁**来实现

### InnoDB MVCC 实现原理

`InnoDB` 中 `MVCC` 的实现方式为：每一行记录都有两个隐藏列：`DATA_TRX_ID`、`DATA_ROLL_PTR`（如果没有主键，则还会多一个隐藏的主键列）。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g2ls3vmt1zj20lg03ewej.jpg)

#### DATA_TRX_ID 

记录最近更新这条行记录的`事务 ID`，大小为 `6` 个字节

#### DATA_ROLL_PTR

表示指向该行回滚段`（rollback segment）`的指针，大小为 `7` 个字节，`InnoDB` 便是通过这个指针找到之前版本的数据。该行记录上所有旧版本，在 `undo` 中都通过链表的形式组织。

#### DB_ROW_ID

行标识（隐藏单调自增 `ID`），大小为 `6` 字节，如果表没有主键，`InnoDB` 会自动生成一个隐藏主键，因此会出现这个列。另外，每条记录的头信息（`record header`）里都有一个专门的 `bit`（`deleted_flag`）来表示当前记录是否已经被删除。

### 如何组织 Undo Log 链

> 关于 Redo Log 和 Undo Log 的相关概念可见之前的文章 [InnoDB 中的 redo 和 undo log](https://chenjiayang.me/2019/04/13/mysql-innodb-redo-undo/)

上文提到，在多个事务并行操作某行数据的情况下，不同事务对该行数据的 UPDATE 会产生多个版本，然后通过回滚指针组织成一条 `Undo Log` 链，这节我们通过一个简单的例子来看一下 `Undo Log` 链是如何组织的，`DATA_TRX_ID` 和 `DATA_ROLL_PTR` 两个参数在其中又起到什么样的作用。

还是以上文 `MVCC` 的例子，事务 `A` 对值 `x` 进行更新之后，该行即产生一个新版本和旧版本。假设之前插入该行的事务 `ID` 为 `100`，事务 `A` 的 `ID` 为 `200`，该行的隐藏主键为 `1`。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g2lsdooktaj20o808odg5.jpg)

事务 `A` 的操作过程为：

1. 对 `DB_ROW_ID = 1` 的这行记录加排他锁
2. 把该行原本的值拷贝到 `undo log` 中，`DB_TRX_ID` 和 `DB_ROLL_PTR` 都不动
3. 修改该行的值这时产生一个新版本，更新 `DATA_TRX_ID` 为修改记录的事务 `ID`，将 `DATA_ROLL_PTR` 指向刚刚拷贝到 `undo log` 链中的旧版本记录，这样就能通过 `DB_ROLL_PTR` 找到这条记录的历史版本。如果对同一行记录执行连续的 `UPDATE`，`Undo Log` 会组成一个链表，遍历这个链表可以看到这条记录的变迁
4. 记录 `redo log`，包括 `undo log` 中的修改

那么 `INSERT` 和 `DELETE` 会怎么做呢？其实相比 `UPDATE` 这二者很简单，`INSERT` 会产生一条新纪录，它的 `DATA_TRX_ID` 为当前插入记录的事务 `ID`；`DELETE` 某条记录时可看成是一种特殊的 `UPDATE`，其实是软删，真正执行删除操作会在 `commit` 时，`DATA_TRX_ID` 则记录下删除该记录的事务 `ID`。

### 如何实现一致性读 —— ReadView

在 `RU` 隔离级别下，直接读取版本的最新记录就 OK，对于 `SERIALIZABLE` 隔离级别，则是通过加锁互斥来访问数据，因此不需要 `MVCC` 的帮助。因此 `MVCC` 运行在 `RC` 和 `RR` 这两个隔离级别下，当 `InnoDB` 隔离级别设置为二者其一时，在 `SELECT` 数据时就会用到版本链

> 核心问题是版本链中哪些版本对当前事务可见？

`InnoDB` 为了解决这个问题，设计了 `ReadView`（可读视图）的概念。

#### RR 下的 ReadView 生成

在 `RR` 隔离级别下，每个事务 `touch first read` 时（本质上就是执行第一个 `SELECT` 语句时，后续所有的 `SELECT` 都是复用这个 `ReadView`，其它 `update`, `delete`, `insert` 语句和一致性读 `snapshot` 的建立没有关系），会将当前系统中的所有的活跃事务拷贝到一个列表生成` ReadView`。

下图中事务 `A` 第一条 `SELECT` 语句在事务 `B` 更新数据前，因此生成的 `ReadView` 在事务 `A` 过程中不发生变化，即使事务 `B` 在事务 `A` 之前提交，但是事务 `A` 第二条查询语句依旧无法读到事务 `B` 的修改。

![](http://ww1.sinaimg.cn/large/c3beb895ly1g49zfpi9jqj20h90gz0tp.jpg)

下图中，事务 `A` 的第一条 `SELECT` 语句在事务 `B` 的修改提交之后，因此可以读到事务 `B` 的修改。**但是注意，如果事务 `A` 的第一条 `SELECT` 语句查询时，事务 `B` 还未提交，那么事务 `A` 也查不到事务 `B` 的修改。**

![](http://ww1.sinaimg.cn/large/c3beb895ly1g49zqz1t94j20h90gz755.jpg)

#### RC 下的 ReadView 生成

在 `RC` 隔离级别下，每个 `SELECT` 语句开始时，都会重新将当前系统中的所有的活跃事务拷贝到一个列表生成 `ReadView`。二者的区别就在于生成 `ReadView` 的时间点不同，一个是事务之后第一个 `SELECT` 语句开始、一个是事务中每条 `SELECT` 语句开始。

`ReadView` 中是当前活跃的事务 `ID` 列表，称之为 `m_ids`，其中最小值为 `up_limit_id`，最大值为 `low_limit_id`，事务 `ID` 是事务开启时 `InnoDB` 分配的，其大小决定了事务开启的先后顺序，因此我们可以通过 `ID` 的大小关系来决定版本记录的可见性，具体判断流程如下：

1. 如果被访问版本的 `trx_id` 小于 `m_ids` 中的最小值 `up_limit_id`，说明生成该版本的事务在 `ReadView` 生成前就已经提交了，所以该版本可以被当前事务访问。

2. 如果被访问版本的 `trx_id` 大于 `m_ids` 列表中的最大值 `low_limit_id`，说明生成该版本的事务在生成 `ReadView` 后才生成，所以该版本不可以被当前事务访问。需要根据 `Undo Log` 链找到前一个版本，然后根据该版本的 DB_TRX_ID 重新判断可见性。

3. 如果被访问版本的 `trx_id` 属性值在 `m_ids` 列表中最大值和最小值之间（包含），那就需要判断一下 `trx_id` 的值是不是在 `m_ids` 列表中。如果在，说明创建 `ReadView` 时生成该版本所属事务还是活跃的，因此该版本不可以被访问，需要查找 Undo Log 链得到上一个版本，然后根据该版本的 `DB_TRX_ID` 再从头计算一次可见性；如果不在，说明创建 `ReadView` 时生成该版本的事务已经被提交，该版本可以被访问。

4. 此时经过一系列判断我们已经得到了这条记录相对 `ReadView` 来说的可见结果。此时，如果这条记录的 `delete_flag` 为 `true`，说明这条记录已被删除，不返回。否则说明此记录可以安全返回给客户端。

![](https://liuzhengyang.github.io/images/readview-visible.jpg)

#### 举个例子

#### RC 下的 MVCC 判断流程

我们现在回看刚刚的查询过程，为什么事务 `B` 在 `RC` 隔离级别下，两次查询的 `x` 值不同。`RC` 下 `ReadView` 是在语句粒度上生成的。

当事务 `A` 未提交时，事务 `B` 进行查询，假设事务 `B` 的事务 `ID` 为 `300`，此时生成 `ReadView` 的 `m_ids` 为 [200，300]，而最新版本的 `trx_id` 为 `200`，处于 `m_ids` 中，则该版本记录不可被访问，查询版本链得到上一条记录的 trx_id 为 `100`，小于 `m_ids` 的最小值 `200`，因此可以被访问，此时事务 `B` 就查询到值 `10` 而非 `20`。

待事务 `A` 提交之后，事务 `B` 进行查询，此时生成的 `ReadView` 的 `m_ids` 为 [300]，而最新的版本记录中 `trx_id` 为 `200`，小于 `m_ids` 的最小值 `300`，因此可以被访问到，此时事务 `B` 就查询到 `20`。

#### RR 下的 MVCC 判断流程

如果在 `RR` 隔离级别下，为什么事务 `B` 前后两次均查询到 `10` 呢？`RR` 下生成 `ReadView` 是在事务开始时，m_ids 为 [200,300]，后面不发生变化，因此即使事务 `A` 提交了，`trx_id` 为 `200` 的记录依旧处于 `m_ids` 中，不能被访问，只能访问版本链中的记录 `10`。

#### 一个争论点

其实并非所有的情况都能套用 `MVCC` 读的判断流程，**特别是针对在事务进行过程中，另一个事务已经提交修改的情况下**，这时不论是 `RC` 还是 `RR`，直接套用 `MVCC` 判断都会有问题，例如 `RC` 下：

![](http://ww1.sinaimg.cn/large/c3beb895ly1g49yvo88rsj20h90gzwf4.jpg)

事务 `A` 的 `trx_id = 200`，事务 `B` 的 `trx_id = 300`，且事务 `B` 修改了数据之后在事务 `A` 之前提交，此时 `RC` 下事务 `A` 读到的数据为事务 `B` 修改后的值，这是很显然的。下面我们套用下 `MVCC` 的判断流程，考虑到事务 `A` 第二次 `SELECT` 时，`m_ids` 应该为 [200]，此时该行数据最新的版本 `DATA_TRX_ID = 300` 比 `200` 大，照理应该不能被访问，但实际上事务 `A` 选取了这条记录返回。

这里其实应该结合 `RC` 的本质来看，`RC` 的本质就是事务中每一条 `SELECT` 语句均可以看到其他已提交事务对数据的修改，那么只要该事物已经提交其结果就是可见的，与这两个事务开始的先后顺序无关，**不完全适用于 MVCC 读**。

`RR` 级别下还是用之前那张图：

![](http://ww1.sinaimg.cn/large/c3beb895ly1g49zqz1t94j20h90gz755.jpg)

这张图的流程中，事务 `B` 的 `trx_id = 300` 比事务 `A` `200` 小，且事务 `B` 先于事务 `A` 提交，按照 `MVCC` 的判断流程，事务 `A` 生成的 `ReadView` 为 [200]，最新版本的行记录 `DATA_TRX_ID = 300` 比 `200` 大，照理不能访问到，但是事务 `A` 实际上读到了事务 `B` 已经提交的修改。这里还是结合 `RR` 本质进行解释，`RR` 的本质是从第一个 `SELECT` 语句生成 `ReadView` 开始，任何已经提交过的事务的修改均可见。

### 总结

`RC`、`RR` 两种隔离级别的事务在执行普通的读操作时，通过访问版本链的方法，使得事务间的读写操作得以并发执行，从而提升系统性能。`RC`、`RR` 这两个隔离级别的一个很大不同就是生成 `ReadView` 的时间点不同，`RC` 在每一次 `SELECT` 语句前都会生成一个 `ReadView`，事务期间会更新，因此在其他事务提交前后所得到的 `m_ids` 列表可能发生变化，使得先前不可见的版本后续又突然可见了。而 `RR` 只在事务的第一个 `SELECT` 语句时生成一个 `ReadView`，事务操作期间不更新。

### 参考资料
- 《高性能 MySQL》
- [Understanding InnoDB MVCC](http://ronaldbradford.com/blog/understanding-innodb-mvcc-2009-07-15/)
- [15.3 InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [MySQL · 引擎特性 · InnoDB MVCC 相关实现](http://mysql.taobao.org/monthly/2018/11/04/)
- [MySQL事务隔离级别和MVCC](https://juejin.im/post/5c9b1b7df265da60e21c0b57)
- [MVCC 原理探究及 MySQL 源码实现分析](https://juejin.im/entry/58f86815ac502e00638e1c97)
- [深入浅出INNODB MVCC机制与原理](https://wenku.baidu.com/view/69d5c129192e45361066f5fb.html)




