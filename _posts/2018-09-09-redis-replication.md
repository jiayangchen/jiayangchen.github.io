---
layout: post
title: "Redis 主从复制 psync1 和 psync2 的区别"
subtitle: "Redis 2.8 版本带来了 psync1，解决了断线重连之后的全量同步问题，4.0 版本带来了其改进版 psync2，解决了主从切换和 slave 重启导致的全量同步问题。"
date: 2018-09-09
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - Redis
---

## 写在前面
在分布式环境中，数据副本 (Replica) 和复制 (Replication) 作为提升系统可用性和读写性能的有效手段被大量应用系统设计中，Redis 也不例外。Redis 作为单机数据库使用时，适用常见有限且存在单点宕机问题，无法维持高可用。因此 Redis 允许通过 SLAVEOF 命令或者 slaveof 配置项来让一个 Redis server 复制另一个 Redis server 的数据集和状态，我们称之为主从复制，主服务器下文称 `master`，从服务器下文称 `slave`，Redis 采用异步的复制机制。

## 主从复制机制的演变
从 Redis 2.6 到 4.0 开发人员对复制流程进行逐步的优化，以下是演进过程：

* 2.8 版本之前 Redis 复制采用 sync 命令，无论是第一次主从复制还是断线重连后再进行复制都采用全量同步，成本高
* 2.8 ~ 4.0 之间复制采用 psync 命令，这一特性主要添加了 Redis 在断线重连时候可通过 offset 信息使用部分同步
* 4.0 版本之后也采用 psync，相比于 2.8 版本的 psync 优化了增量复制，这里我们称为 psync2，2.8 版本的 psync 可以称为 psync1

我们先介绍 psync1 的实现，再介绍 psync2 的优化点，至于旧版 sync 的机制本文不再赘述。

## psync1
为了解决旧版 SYNC 在处理断线重连复制场景下的低效问题，Redis 2.8 采用 PSYNC 代替 SYNC 命令。PSYNC 命令具有全量同步和部分同步两种模式。

### 全量重同步
前者和 SYNC 大致相同，都是让 `master` 生成并发送 RDB 文件，然后再将保存在缓冲区中的写命令传播给 `slave` 来进行同步，相当于只有同步和命令传播两个阶段。

### 部分重同步
部分同步适用于断线重连之后的同步，`slave` 只需要接收断线期间丢失的写命令就可以，不需要进行全量同步。为了实现部分同步，引入了复制偏移量`（offset）`、复制积压缓冲区`（replication backlog buffer）`和运行 ID `（run_id）`三个概念。

#### 复制偏移量
执行主从复制的双方都会分别维护一个复制偏移量，`master` 每次向 `slave` 传播 N 个字节，自己的复制偏移量就增加 N；同理 `slave` 接收 N 个字节，自身的复制偏移量也增加 N。通过对比主从之间的复制偏移量就可以知道主从间的同步状态。

#### 复制积压缓冲区
复制积压缓冲区是 `master` 维护的一个固定长度的 FIFO 队列，默认大小为 1MB。当 `master` 进行命令传播时，不仅将写命令发给 `slave` 还会同时写进复制积压缓冲区，因此 `master` 的复制积压缓冲区会保存一部分最近传播的写命令。当 `slave` 重连上 `master` 时会将自己的复制偏移量通过 PSYNC 命令发给 `master`，`master` 检查自己的复制积压缓冲区，如果发现这部分未同步的命令还在自己的复制积压缓冲区中的话就可以利用这些保存的命令进行部分同步，反之如果断线太久这部分命令已经不在复制缓冲区中了，那没办法只能进行全量同步。

#### 运行 ID
令人疑惑的是上述逻辑看似已经很圆满了，这个 `run_id` 是做什么用呢？其实这是因为 `master` 可能会在 `slave` 断线期间发生变更，例如可能超时失去联系或者宕机导致断线重连的是一个崭新的 master，不再是断线前复制的那个了。自然崭新的 `master` 没有之前维护的复制积压缓冲区，只能进行全量同步。因此每个 Redis server 都会有自己的运行 ID，由 40 个随机的十六进制字符组成。当 `slave` 初次复制 `master` 时，master 会将自己的运行 ID 发给 `slave` 进行保存，这样 `slave`

重连时再将这个运行  ID 发送给重连上的 `master` ，master 会接受这个 ID 并于自身的运行 ID 比较进而判断是否是同一个 master。

### psync1 流程
如果 `slave` 以前没有复制过任何 master，或者之前执行过 SLAVEOF NO ONE 命令，那么 `slave` 在开始一次新的复制时将向主服务器发送 PSYNC ? -1 命令，主动请求 `master` 进行完整重同步（因为这时不可能执行部分重同步）。相反地，如果 `slave` 已经复制过某个 master，那么 `slave` 在开始一次新的复制时将向 `master` 发送 PSYNC runid offset 命令：其中 runid 是上一次复制的 `master` 的运行ID，而 offset 则是 `slave` 当前的复制偏移量，接收到这个命令的 `master` 会通过这两个参数来判断应该对 `slave` 执行哪种同步操作。

根据情况，接收到 PSYNC 命令的 `master` 会向 `slave` 返回以下三种回复的其中一种：

如果 `master` 返回 +FULLRESYNC runid offset 回复，那么表示 `master` 将与 `slave` 执行完整重同步操作：其中 runid 是这个 `master` 的运行 ID，slave 会将这个 ID 保存起来，在下一次发送 PSYNC 命令时使用；而 offset 则是 `master` 当前的复制偏移量，slave 会将这个值作为自己的初始化偏移量

如果 `master` 返回 +CONTINUE 回复，那么表示 `master` 将与 `slave` 执行部分同步操作，slave 只要等着 `master` 将自己缺少的那部分数据发送过来就可以了

如果 `master` 返回 -ERR 回复，那么表示 `master` 的版本低于 Redis 2.8，它识别不了 psync 命令，slave 将向 `master` 发送 SYNC 命令，并与 `master` 执行完整同步操作

由此可见 psync 也有不足之处，当 `slave` 重启以后 `master` runid 发生变化，也就意味者 `slave` 还是会进行全量复制，而在实际的生产中进行 `slave` 的维护很多时候会进行重启，而正是有由于全量同步需要 `master` 执行快照，以及数据传输会带不小的影响。因此在 4.0 版本，psync 命令做了改进，我们称之为 psync2。

## psync2
Redis 4.0 版本新增 混合持久化，还优化了 psync（以下称 psync2），psync2 最大的变化支持两种场景下的部分重同步，一个场景是 `slave` 提升为 `master` 后，其他 `slave` 可以从新提升的 `master` 进行部分重同步，这里需要 `slave` 默认开启复制积压缓冲区；另外一个场景就是 `slave` 重启后，可以进行部分重同步。这里要注意和 psync1 的运行 ID 相比，这里的复制 ID 有不一样的意义。

### 优化细节
Redis 4.0 引入另外一个变量 master_replid 2 来存放同步过的 `master` 的复制 ID，同时复制 ID 在 `slave` 上的意义不同于之前的运行 ID，复制 ID 在 `master` 的意义和之前运行 ID 仍然是一样的，但对于 `slave` 来说，它保存的复制 ID（即 replid） 表示当前正在同步的 `master` 的复制 ID 。master_replid 2  则表示前一个 `master` 的复制 ID（如果它之前没复制过其他的 master，那这个字段没用），这个在主从角色发生改变的时候会用到。

```c
struct redisServer {
    ...
    /* Replication (`master`) */                                        
    char replid[CONFIG_RUN_ID_SIZE+1];  /* My current replication ID. */
    char replid2[CONFIG_RUN_ID_SIZE+1]; /* replid inherited from `master`*/ 
}
```

`slave` 在意外关闭前会调用 rdbSaveInfoAuxFields 函数把当前的复制 ID（即关闭前正在复制的 `master` 的 replid，因为 `slave` 中的 replid 字段保存的是 `master` 的复制 ID） 和复制偏移量一起保存到 RDB 文件中，后面该 `slave` 重启的时候，就可以从 RDB 文件中读取复制 ID 和复制偏移量，然后重连上 `master` 后 `slave` 将这两个值发送给 master，master 会如下判断是否允许 psync：

```c
// 如果 `slave` 发送过来的复制 ID 是当前 `master` 的复制 ID, 说明 `master` 没变过
    if (strcasecmp(master_replid, server.replid) &&   
        // 或者和现在的新 `master` 曾经属于同一 `master`
        (strcasecmp(master_replid, server.replid2) ||      
         // 但同步进度不能比当前 `master` 还快
         psync_offset > server.second_replid_offset)) {
      	... ...
    }

    // 判断同步进度是否已经超过范围
    if (!server.repl_backlog ||                                                        
        psync_offset < server.repl_backlog_off ||                                      
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen)) {                                                                                  
        ... ...
    }  
```

另外当节点从 `slave` 提升为 `master` 后，会保存两个复制 ID（之前角色是 `slave` 的时候 replid2 没用，现在要派上用场了），分别是 replid 和 replid 2，其他 `slave` 复制的时候可以根据第二个复制 ID 来进行部分重同步。对应上述代码中第二行判断的情况。

## 参考资料
* [Redis Replication](https://redis.io/topics/replication)
* [Understanding edis Replication](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Replication.Redis.Groups.html)
* [The first release candidate of Redis 4.0 is out](http://antirez.com/news/110)

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>



