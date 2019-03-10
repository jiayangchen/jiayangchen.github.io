---
layout: post
title: "黎明号角 —— MySQL InnoDB 存储引擎"
subtitle: "MySQL 得益于开放的可插拔设计，允许替换不同的底层存储引擎，InnoDB 就是其中涌现的代表，自 5.5.8 以来已经成为 MySQL 的默认存储引擎"
date: 2019-03-09
author: "ChenJY"
header-img: "img/mysql.jpg"
catalog: true
tags: 
    - MySQL
---

### InnoDB 存储引擎

`MySQL` 得益于开放的可插拔设计，允许替换不同的底层存储引擎，`InnoDB` 就是其中的代表，最初由第三方公司开发后被 `Oracle` 收购，是 `OLTP` 场景下核心表的首选存储引擎，自 `5.5` 以来已经成为 `MySQL` 的默认存储引擎。

`InnoDB` 是一个健壮的事务型存储引擎，这种存储引擎已经被很多互联网公司使用，为用户操作非常大的数据存储提供了一个强大的解决方案。`MySQL 5.5` 之后，`InnoDB` 就是作为默认的存储引擎。`InnoDB` 还引入了行级锁定和外键约束，在以下场合下，使用 `InnoDB` 是最理想的选择：

1. 更新密集的表。`InnoDB` 存储引擎特别适合处理多重并发的更新请求。
2. 支持事务。`InnoDB` 存储引擎是支持 ACID 事务的标准 MySQL 存储引擎。
3. 自动灾难恢复。与其它存储引擎不同，InnoDB 表能够自动从灾难中恢复。
4. 需要外键。`MySQL` 支持外键的存储引擎只有 `InnoDB`。
5. 支持自动增加列 `AUTO_INCREMENT` 属性。

一般来说，如果需要事务支持，并且有较高的并发读取频率，`InnoDB` 是不错的选择。

![](http://ww1.sinaimg.cn/large/c3beb895gy1g0wluwtaycj22513wbb29.jpg)

### 参考资料

- 《MySQL 技术内幕 InnoDB 存储引擎》第二版 姜承尧著

### 许可协议

- 本文遵守创作共享 [CC BY-NC-SA 3.0协议](https://creativecommons.org/licenses/by-nc-sa/3.0/cn/)