---
layout: post
title: "小议 Redis 的主从复制功能"
subtitle: ""
date: 2018-01-05
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - Redis
---

如何在分布式环境下保持不同实例间的数据一致性，是一个难度极大、经久不衰的话题。常见的 MySQL 中为了维持高可用，采用主从配置的方法，Redis 中也有类似的维持一主多从的方式提高 Redis 集群的高可用性的方案，而其中不可避免的则是如何保证主从实例间的数据一致性，`复制（Replication）`是其解决办法，但是复制是怎么开始的？怎么容错的？复制过程中新进的写操作如何处理等，都是需要仔细考虑的问题，本文就 Redis 复制的方法进行讨论。

### 版本区别
`Redis 2.8` 之前的版本和之后的版本复制功能采用的是不一样的实现原理，主要为了规避 2.8 之前的一旦遇到网络断线重连的问题，从数据库需要全量同步主造成的效率低下问题。虽然目前应用 Redis 2.8 的实际场景较少，基本上都是 3.X 之后了，但是新版的改进是基于旧版的，因此本文还是先讨论旧版实现，之后添加对新版的讲解。

### 事前思考
**我们先想一想如果要在两个 Redis 实例间进行数据同步，会怎么实现呢？** 我想，先要确定主实例（下文简称主）和目标实例（下文简称从）；之后由需要进行复制的从通知主：

> “我要开始复制了，准备好了吗？”

> 主答曰 “准备好了，随时可以开始。”

之后主开始将数据打包，传送给从，从接到数据包之后，更新自己的状态；此时如果主还有新的写操作（不仅仅指字面上的新增操作，还有删除、修改等操作），那么需要存下来，并和已经传给从的数据区分开来，防止之后重复传送；从也需要通知主是否成功收到数据包、已更新多少、是否更新完成等状态；最后，主将新的写操作传给从，这里大家想象一下，如果一直有写操作的话，必定主和从之间需要反复传送这部分写操作，虽然期间彼此数据有差异，但是差异会逐渐减小，最后到达单个命令的差异，此时复制功能也就完成了。

### 具体实现
我们思考了这么多，其实整个流程是很简单直观的，接下去就是如何实现好的问题了。Redis 文档中详细阐述了，Redis 的复制功能分为`同步（sync）`和`命令传播（command propagate）`两个阶段，具体来说同步指的是主将自己的现有状态复制给从的过程；而命令传播指的是之后一旦主发生新的状态修改之后，需要及时告知从的过程。

#### 同步
当需要从去复制主的时候，从会首先执行同步操作（因为这时候一般主从间的数据差异较大）

* 从会向主发送 SYNC 命令
* 主收到之后执行 BGSAVE 命令，在后台生成一个 RDB 文件，并开始将这个时间点时候的写操作存储在缓冲区中
* 主将 RDB 文件发送给从，从载入这个文件，将自己的状态更新完成，此时从跟主之间的差异就在于主新增的操作了
* 主将自己新增的操作也发生给从，从逐步更新完成，愈发接近主的实际状态

#### 命令传播
这之后，从跟主之间已经完成同步，但是如果主接受写操作，那么瞬间二者又会不一致，主需要通过命令传播的方式将新的写命令传递给从，此时的更新量已经较少，不需要依靠 RDB 文件进行大量的同步，而只需要依靠命令为单位就可以。

### 继续思考
这里还有个指的我们思考的问题，那就是如果主从复制期间网络断线了怎么办？这时如果重连，那么究竟是从重新全量同步主的状态，直接舍弃之前已经同步过的数据；还是采用新的方法记录断线前已经同步完的位置，只要重连之后同步新的操作就可以？实际上前者是旧版的复制功能，后者就是新版的改进。

### 新版改进
新版本中 `PSYNC` 命令替代了单纯的 `SYNC` ，其中 `PSYNC` 具有全量重同步和部分重同步两种模式，其中完整重同步用于处理初次复制的情况，类似于之前旧版的 `SYNC` ；而部分重同步则是用于处理断线重连的情况，只需要同步新的更改，采用 `CONTINUE` 标记是否进行部分重同步，提高了效率。

###  新版实现
这里主要讨论部分重同步的具体实现方案了。主要有以下三部分构成：

* 主和从的复制偏移量，表示二者分别进行到哪里了，主侧每发送 `N` 个字节的数据，自身维护的 `offset + N` ； 从侧每次收到 `N` 个字节的数据，自身的 `offset + N` 。由此在重连之后，从主要上报一下自己的 offset ，主就可以知道从和主之间差了多少数据。
* 主的缓冲区，表示断线期间的新增操作，是由主维护的一个 `FIFO` 的队列，默认大小为 `1 MB` 。 当进行命令传播操作时，主会将传播的命令写进缓冲区，这里如果主收到了从断线重连之后上报的 offset ，会先看看缺少的数据是否还存在缓冲区中，若存在则恢复从进行部分重同步；反之进行全量重同步。
* 服务器的运行 ID 用于标识服务器身份。启动时自动生成，由 40 个随机的十六进制字符组成，在进行初次复制时从会保存主传过来的自身的运行 ID ，表示从是具体同步哪个主，一旦发生断线重连，那么需要检测重连上的主是否是之前的主，如果是才决定接下来是部分重同步还是全量重同步；如果重连上的已经不是之间的主了，那么必须全量重同步。

### 参考资料
* <a href="https://redis.io/topics/replication" target="_blank"><b>Redis Replication</b></a>
* <a href="http://product.dangdang.com/23501734.html" target="_blank"><b>《Redis 设计与实现》</b></a> 黄建宏 著

### License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@heapdump?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Patrick Lindenberg"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Patrick Lindenberg</span></a>