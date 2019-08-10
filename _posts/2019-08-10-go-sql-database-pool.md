---
layout: post
title: "借 Go 语言 database/sql 包谈数据库驱动和连接池设计"
subtitle: ""
date: 2019-08-10
author: "ChenJY"
header-img: "img/newblogbg.jpg"
catalog: true
tags: 
    - Golang
---

> 1. 这是公众号的第 xx 篇文章
> 2. 即使你不了解 Go 语言，阅读本文也不会有障碍

- [什么是池化技术](#什么是池化技术)
- [database/sql 包](#databasesql-包)
    - [设计哲学](#设计哲学)
    - [极简接口](#极简接口)
    - [调用关系](#调用关系)
- [连接池设计](#连接池设计)
    - [sql.DB 对象关键属性](#sqldb-对象关键属性)
    - [建立连接](#建立连接)
    - [释放连接](#释放连接)
    - [清理连接](#清理连接)
- [总结](#总结)

### 什么是池化技术
池化技术 (`Pool`) 是一种很常见的编程技巧，在请求量大时能明显优化应用性能，降低系统频繁建连的资源开销。我们日常工作中常见的有数据库连接池、线程池、内存对象池等，它们的特点都是将 “昂贵的”、“费时的” 连接资源维护在一个特定的 “池子” 中，规定其最小连接数、最大连接数、阻塞队列等配置，方便进行统一管理和复用，通常还会附带一些探活机制、强制回收、监控一类的配套功能。

### database/sql 包
#### 设计哲学
在 `Go` 语言中对数据库进行操作需要借助标准库下的 `database/sql` 包进行，它对上层应用提供了标准的 `API` 操作接口，对下层驱动暴露了简单的驱动接口，并在内部实现了连接池管理。这意味着不同数据库的驱动可以很方便地实现这些驱动接口，但不再需要关心连接池的细节，只需要基于单个连接。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5uhfe7w5gj20dk0qa75g.jpg)

#### 极简接口
它对外暴露的接口简单易懂，利于第三方 `Driver` 去实现，接口的功能包括 `Driver` 注册、`Conn`、`Stmt`、`Tx`、`Rows`结果集等，我们通过 `Conn` 和 `Stmt` 这两个接口来体会一下接口设计的精妙（这两个接口对应到 `Java` 就是 `Connection` 和 `Statement` 接口，只是 `Go` 更加简单）

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5uhsboalij21a80fmwgo.jpg)
![](http://ww1.sinaimg.cn/large/c3beb895gy1g5uhwokq5pj21bg0iwtbo.jpg)

我相信你即使没有学习过 `Go` 语言，仅凭你的 `Java` 知识，也可以毫不费力地看懂上面这些接口的意思，这些对于驱动层暴露的接口非常简单，让驱动程序可以方便地去实现。

#### 调用关系
整个 `database/sql` 驱动接口的调用关系非常清晰，简单来说驱动程序先通过 `Open` 方法拿到一个新建的 `Conn`，然后调用 `Conn` 的 `Prepare` 方法，传入 `SQL` 语句得到该语句的 `Stmt`，最后调用 `Stmt` 的 `Exec` 方法传入参数返回结果，查询语句同理，但返回的是行数据。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5uifsyk3hj21du0eu40k.jpg)

### 连接池设计
#### sql.DB 对象关键属性
`Go` 语言操作数据库时，我们先使用 `sql.Open` 方法返回一个具体的 `sql.DB` 对象，如下代码片中的 `db` ：

```go
func main() {
    db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/test")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
}
```

`sql.DB` 对象即是我们访问数据库的入口，我们看看它里面的关键属性，均与连接池的设计相关

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5urt47ugxj20yo0rojw7.jpg)

#### 建立连接
事实上，连接并不是在 `sql.Open` 返回 `db` 对象时就建立的，这一步仅仅开了个接收建连请求的 `channel`，实际建连步骤要等到执行具体 `SQL` 语句时才会进行。下面我们通过一些例子讲述一下连接是怎么建立的，连接池的逻辑又是怎么实现的。

> 讲述这部分原理不会贴太多的源码，那就变成源码解析了，对不了解 Go 语言的同学也不友好，主要希望能传达一些连接池设计的思想。

在 `database/sql` 对上层应用暴露的操作接口中，比较常用的是 `Exec` 和 `Query`，前者常用于执行写 `SQL`，后者可以用于读 `SQL`。但是不论走哪个方法，都会调用到建连逻辑 `db.conn` 方法，附带建连上下文和建连策略两个参数。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5usc3e3ukj20x20tkdit.jpg)

其中建连策略分为 `cachedOrNewConn` 和 `alwaysNewConn`。前者优先从 `freeConn` 空闲连接中取出连接，否则就新建一个；后者则永远走新建连接的逻辑。

使用 `cachedOrNewConn` 策略的建立逻辑中，会先判断是否有空闲连接，如果有取出空闲队列的第一个，紧接着判断该连接是否过期需要被回收，如果没有过期则可以正常使用进入后续逻辑。如果没有空闲连接了，则得判断是不是连接数是不是已经达到最大数目，没有自然可以新建连接，反之就得阻塞这个请求让它等待可用连接了。

如果需要新建连接，则调用底层 `Driver` 实现的连接器的 `Connect` 接口，这部分就是由各个数据库 `Driver` 自行去实现了。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5usuwiak8j20nk0y4q5q.jpg)

#### 释放连接
某个连接使用完毕之后需要归还给连接池，这也是数据库连接池实现中比较重要的逻辑，通常还伴随着对连接的可靠性检测，如果连接异常关闭，那么不应该继续还给连接池，而是应该新建一个连接进行替换。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g5ut4z86cjj20n00ow767.jpg)

在 `Java` 中 `Druid` 连接池会有 `testOnReturn` 或者 `testOnBorrow` 选项，表示在归还连接或者是获取连接时进行有效性检测，但是开启这两项本质上会延长连接被占用的时间，损失一部分性能。`Go` 语言中对这项功能的实现比较简单，并没有具体的有效性检测机制，只是直接根据连接附带的 `err` 信息，如果是 `ErrBadConn` 异常则关闭并发送信号新建一个。

#### 清理连接
`database/sql` 包下提供了与连接池相关的三个关键参数设置，分别是 `maxIdle`、`maxOpen` 和 `maxLifeTime`。

> 三个参数的含义很容易理解，如果想要深入了解，推荐阅读 [Configuring sql.DB for Better Performance](https://www.alexedwards.net/blog/configuring-sqldb)

`MySQL` 侧会强制 `kill` 掉长时间空闲的连接（8h），`Go` 语言提供了 `maxLifeTime` 选项设置连接被复用的最大时间，注意并不是连接空闲时间，而是从连接建立到这个时间点就会被回收，从而保证连接活性。

这块的清理机制是通过一个异步任务来做的，关键是逻辑是每个一秒遍历检查 `freeConn` 中的空闲连接，判断是否超出最大复用期限，超出的连接加入 `Closing` 数组，后续被 `Close`。

### 总结
最近的工作内容是基于 `go-sql-driver` 实现了一个支持读写分离和高可用的自定义 `driver`，在调研和学习期间感受到了 `Go` 语言 `database/sql` 包的简明清晰，虽然它在部分功能的实现上偏简单甚至没有，但是依旧覆盖了大部分数据库连接池的主要功能和特性，因此我觉得用它来抛砖引玉是个好选择。


