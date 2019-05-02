---
layout: post
title: "MySQL InnoDB MVCC 机制的原理及实现"
subtitle: "多版本并发控制，是现代数据库引擎实现中常用的处理读写冲突的手段，目的在于提高数据库高并发场景下的吞吐性能。"
date: 2019-05-01
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - MySQL
---

### 什么是 MVCC

`MVCC (Multiversion Concurrency Control)` 中文全程叫**多版本并发控制**，是现代数据库（包括 `MySQL`、`Oracle`、`PostgreSQL` 等）引擎实现中常用的处理读写冲突的手段，**目的在于提高数据库高并发场景下的吞吐性能**。

如此一来，不同事务并发过程中，`SELECT` 操作可以不加锁而是通过 `MVCC` 机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别，从而解决并发场景下的读写冲突。

下面举一个多版本读的例子，例如两个事务 `A` 和 `B` 按照如下顺序进行更新和读取操作

![](http://ww1.sinaimg.cn/large/c3beb895gy1g2lrq1k5r0j20iy0hkt9l.jpg)

在事务 `A` 提交前后，事务 `B` 读取到的 `x` 的值是什么呢？答案是：事务 `B` 在不同的隔离级别下，读取到的值不一样。

1. 如果事务 `B` 的隔离级别是读未提交（RU），那么两次读取均读取到 `x` 的最新值，即 `20`。
2. 如果事务 `B` 的隔离级别是读已提交（RC），那么第一次读取到旧值 `10`，第二次因为事务 `A` 已经提交，则读取到新值 20。
3. 如果事务 `B` 的隔离级别是可重复读或者串行（RR，S），则两次均读到旧值 `10`，不论事务 `A` 是否已经提交。

可见在不同的隔离级别下，数据库通过 `MVCC` 和隔离级别，让事务之间并行操作遵循了某种规则，来保证单个事务内前后数据的一致性。

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

行标识（隐藏单调自增 ID），大小为 `6` 字节，如果表没有主键，`InnoDB` 会自动生成一个隐藏主键，因此会出现这个列。

### 如何组织 Undo Log 链

> 关于 Redo Log 和 Undo Log 的相关概念可见之前的文章 [InnoDB 中的 redo 和 undo log](https://chenjiayang.me/2019/04/13/mysql-innodb-redo-undo/)

上文提到，在多个事务并行操作某行数据的情况下，不同事务对该行数据的修改会产生多个版本，然后通过回滚指针组织成一条 `Undo Log` 链，这节我们通过一个简单的例子来看一下 `Undo Log` 链是如何组织的，`DATA_TRX_ID` 和 `DATA_ROLL_PTR` 两个参数在其中又起到什么样的作用。

还是以上文 `MVCC` 的例子，事务 `A` 对值 `x` 进行更新之后，该行即产生一个新版本和旧版本。假设之前插入该行的事务 `ID` 为 `100`，事务 `A` 的 `ID` 为 `200`，该行的隐藏主键为 `1`。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g2lsdooktaj20o808odg5.jpg)

事务 `A` 的操作过程为：

1. 对 `DB_ROW_ID = 1` 的这行记录加排他锁
2. 把该行原本的值拷贝到 `undo log` 中
3. 修改该行的值，更新 `DATA_TRX_ID`，将 `DATA_ROLL_PTR` 指向刚刚拷贝到 `undo log` 链中的旧版本记录
4. 记录 `redo log`，包括 `undo log` 中的修改

### ReadView

在 `RU` 隔离级别下，直接读取版本的最新记录就 OK，对于 `SERIALIZABLE` 隔离级别，则是通过加锁互斥来访问数据，因此不需要 `MVCC` 的帮助。因此 `MVCC` 运行在 `RC` 和 `RR` 这两个隔离级别下，当 `InnoDB` 隔离级别设置为二者其一时，在 `SELECT` 数据时就会用到版本链

> 核心问题是版本链中哪些版本对当前事务可见？

`InnoDB` 为了解决这个问题，设计了 `ReadView`（可读视图）的概念。在 `RR` 隔离级别下，每个事务开始时，会将当前系统中的所有的活跃事务拷贝到一个列表生成` ReadView`。

在 `RC` 隔离级别下，每个语句开始时，会将当前系统中的所有的活跃事务拷贝到一个列表生成 `ReadView`。二者的区别就在于生成 `ReadView` 的时间点不同，一个是事务开始一个是语句开始。

`ReadView` 中是当前活跃的事务 `ID` 列表，称之为 `m_ids`，事务 `ID` 是事务开启时 `InnoDB` 分配的，其大小决定了事务开启的先后顺序，因此我们可以通过 `ID` 的大小关系来决定版本记录的可见性，具体判断流程如下：

1. 如果被访问版本的 `trx_id` 小于 `m_ids` 中的最小值，说明生成该版本的事务在 `ReadView` 生成前就已经提交了，所以该版本可以被当前事务访问。
2. 如果被访问版本的 `trx_id` 大于 `m_ids` 列表中的最大值，说明生成该版本的事务在生成 `ReadView` 后才生成，所以该版本不可以被当前事务访问。
3. 如果被访问版本的 `trx_id` 属性值在 `m_ids` 列表中最大值和最小值之间（包含），那就需要判断一下 `trx_id` 的值是不是在 `m_ids` 列表中。如果在，说明创建 `ReadView` 时生成该版本所属事务还是活跃的，因此该版本不可以被访问；如果不在，说明创建 `ReadView` 时生成该版本的事务已经被提交，该版本可以被访问。

#### 举个例子

#### RC

我们现在回看刚刚的查询过程，为什么事务 `B` 在 `RC` 隔离级别下，两次查询的 `x` 值不同。`RC` 下 `ReadView` 是在语句粒度上生成的。

当事务 `A` 未提交时，事务 `B` 进行查询，假设事务 `B` 的事务 `ID` 为 `300`，此时生成 `ReadView` 的 `m_ids` 为 [200，300]，而最新版本的 `trx_id` 为 `200`，处于 `m_ids` 中，则该版本记录不可被访问，查询版本链得到上一条记录的 trx_id 为 `100`，小于 `m_ids` 的最小值 `200`，因此可以被访问，此时事务 `B` 就查询到值 `10` 而非 `20`。

待事务 `A` 提交之后，事务 `B` 进行查询，此时生成的 `ReadView` 的 `m_ids` 为 [300]，而最新的版本记录中 `trx_id` 为 `200`，小于 `m_ids` 的最小值 `300`，因此可以被访问到，此时事务 `B` 就查询到 `20`。

#### RR

如果在 `RR` 隔离级别下，为什么事务 `B` 前后两次均查询到 `10` 呢？`RR` 下生成 `ReadView` 是在事务开始时，m_ids 为 [200,300]，后面不发生变化，因此即使事务 `A` 提交了，`trx_id` 为 `200` 的记录依旧处于 `m_ids` 中，不能被访问，只能访问版本链中的记录 `10`。

### InnoDB 可见性判断源码

```c++
// Friend declaration
class MVCC;
/** Read view lists the trx ids of those transactions for which a consistent
read should not see the modifications to the database. */
...
class ReadView {
    ...
public:
    ReadView();
    ~ReadView();
    /** Check whether transaction id is valid.
    @param[in]  id      transaction id to check
    @param[in]  name        table name */
    static void check_trx_id_sanity(
        trx_id_t        id,
        const table_name_t& name);
    /** Check whether the changes by id are visible.
    @param[in]  id  transaction id to check against the view
    @param[in]  name    table name
    @return whether the view sees the modifications of id. */
    bool changes_visible(
        trx_id_t        id,
        const table_name_t& name) const
        MY_ATTRIBUTE((warn_unused_result))
    {
        ut_ad(id > 0);
        // 如果小于 m_ids 最小值或者是事务自身，可见
        if (id < m_up_limit_id || id == m_creator_trx_id) {
            return(true);
        }
        check_trx_id_sanity(id, name);
        // 如果大于 m_ids 最大值，不可见
        if (id >= m_low_limit_id) {
            return(false);
        } 
        // 如果当前活跃事务为空，可见
        else if (m_ids.empty()) {
            return(true);
        }
        const ids_t::value_type*    p = m_ids.data();
        return(!std::binary_search(p, p + m_ids.size(), id));
    }
    
private:
    // Disable copying
    ReadView(const ReadView&);
    ReadView& operator=(const ReadView&);
private:
   // 读操作不应该看到任何 trx_id 比该值大的事务
   // 即 m_ids 中的最大值
    /** The read should not see any transaction with trx id >= this
    value. In other words, this is the "high water mark". */
    trx_id_t    m_low_limit_id;
    // 任何比该值小的事务记录均可对读操作可见
    // 即 m_ids 中的是最小值
    /** The read should see all trx ids which are strictly
    smaller (<) than this value.  In other words, this is the
    low water mark". */
    // 
    trx_id_t    m_up_limit_id;
    /** trx id of creating transaction, set to TRX_ID_MAX for free
    views. */
    trx_id_t    m_creator_trx_id;
    // 生成快照的时候所有活跃的事务 ID 列表
    /** Set of RW transactions that was active when this snapshot
    was taken */
    ids_t       m_ids;
    /** The view does not need to see the undo logs for transactions
    whose transaction number is strictly smaller (<) than this value:
    they can be removed in purge if not needed by other views */
    trx_id_t    m_low_limit_no;
    /** AC-NL-RO transaction view that has been "closed". */
    bool        m_closed;
    typedef UT_LIST_NODE_T(ReadView) node_t;
    /** List of read views in trx_sys */
    byte        pad1[64 - sizeof(node_t)];
    node_t      m_view_list;
};
```

### 总结

`RC`、`RR` 两种隔离级别的事务在执行普通的读操作时，通过访问版本链的方法，使得事务间的读写操作得以并发执行，从而提升系统性能。`RC`、`RR` 这两个隔离级别的一个很大不同就是生成 `ReadView` 的时间点不同，`RC` 在每一次读操作语句前都会生成一个 `ReadView`，事务期间会更新，因此在其他事务提交前后所得到的 `m_ids` 列表可能发生变化，使得先前不可见的版本后续又突然可见了。而 `RR` 只在事务开始前生成一个 `ReadView`，事务操作期间不更新。

### 参考资料
- 《高性能 MySQL》
- [Understanding InnoDB MVCC](http://ronaldbradford.com/blog/understanding-innodb-mvcc-2009-07-15/)
- [15.3 InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [MySQL · 引擎特性 · InnoDB MVCC 相关实现](http://mysql.taobao.org/monthly/2018/11/04/)
- [MySQL事务隔离级别和MVCC](https://juejin.im/post/5c9b1b7df265da60e21c0b57)
- [MVCC 原理探究及 MySQL 源码实现分析](https://juejin.im/entry/58f86815ac502e00638e1c97)




