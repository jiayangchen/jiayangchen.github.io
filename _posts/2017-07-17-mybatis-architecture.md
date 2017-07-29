---
layout: post
title: "【转载】MyBatis的架构设计以及实例分析"
subtitle: ""
date: 2017-07-17
author: "亦山"
header-img: "img/db.jpg"
catalog: true
tags: 
    - MyBatis
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>

最近实习的项目又用到 Mybatis 了，之前一直用 SSM 框架，但是因为时间紧任务重，从没深入研究过其中的原理，今天正好趁着这个机会，看了一篇讲的非常好的博文，作者也说可以在署名之后转载，因此贴上来。

> 下面是正文，[原文链接](http://blog.csdn.net/luanlouis/article/details/40422941),作者：亦山

MyBatis 是目前非常流行的 ORM 框架，它的功能很强大，然而其实现却比较简单、优雅。本文主要讲述 MyBatis 的架构设计思路，并且讨论 MyBatis 的几个核心部件，然后结合一个 select 查询实例，深入代码，来探究 MyBatis 的实现。

### 一、Mybatis的框架设计
![image](http://img.blog.csdn.net/20141028232313593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVhbmxvdWlz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 1. 接口层与数据交互方式
MyBatis和数据库的交互有两种方式：
1. 使用传统的MyBatis提供的API；
2. 使用Mapper接口

##### 使用传统的MyBatis提供的API
这是传统的传递 Statement Id 和查询参数给 SqlSession 对象，使用 SqlSession 对象完成和数据库的交互；MyBatis 提供了非常方便和简单的API，供用户实现对数据库的增删改查数据操作，以及对数据库连接信息和MyBatis 自身配置信息的维护操作。

![image](http://img.blog.csdn.net/20141103155203576)

上述使用 MyBatis 的方法，是创建一个和数据库打交道的 SqlSession 对象，然后根据 Statement Id 和参数来操作数据库，这种方式固然很简单和实用，但是它不符合面向对象语言的概念和面向接口编程的编程习惯。由于面向接口的编程是面向对象的大趋势，MyBatis 为了适应这一趋势，增加了第二种使用MyBatis 支持接口（Interface）调用方式。

##### 使用Mapper接口
MyBatis 将配置文件中的每一个 <mapper> 节点抽象为一个 Mapper 接口，而这个接口中声明的方法和跟 <mapper> 节点中的 <select|update|delete|insert> 节点项对应，即 <select|update|delete|insert> 节点的 id 值为 Mapper 接口中的方法名称，parameterType 值表示Mapper 对应方法的入参类型，而 resultMap 值则对应了 Mapper 接口表示的返回值类型或者返回结果集的元素类型。

![image](http://img.blog.csdn.net/20141103163301421)

根据 MyBatis 的配置规范配置好后，通过 SqlSession.getMapper(XXXMapper.class) 方法，MyBatis 会根据相应的接口声明的方法信息，通过<b>动态代理机制</b>生成一个 Mapper 实例，我们使用 Mapper 接口的某一个方法时，MyBatis 会根据这个方法的方法名和参数类型，确定Statement Id，底层还是通过 SqlSession.select("statementId",parameterObject); 或者 SqlSession.update("statementId",parameterObject); 等等来实现对数据库的操作，（至于这里的动态机制是怎样实现的，我将准备专门一片文章来讨论，敬请关注~）
MyBatis 引用 Mapper 接口这种调用方式，纯粹是为了满足面向接口编程的需要。（其实还有一个原因是在于，面向接口的编程，使得用户在接口上可以使用注解来配置SQL语句，这样就可以脱离XML配置文件，实现“0配置”）。

#### 2.数据处理层
数据处理层可以说是 MyBatis 的核心，从大的方面上讲，它要完成三个功能：
1. 通过传入参数构建动态 SQL 语句；
2. SQL语句的执行以及封装查询结果集成 List<E>

##### 参数映射和动态SQL语句生成
动态语句生成可以说是MyBatis框架非常优雅的一个设计，MyBatis 通过传入的参数值，使用 Ognl 来动态地构造 SQL 语句，使得 MyBatis 有很强的灵活性和扩展性。参数映射指的是对于 Java 数据类型和 jdbc 数据类型之间的转换：这里有包括两个过程：查询阶段，我们要将java 类型的数据，转换成 jdbc 类型的数据，通过 preparedStatement.setXXX() 来设值；另一个就是对 resultset 查询结果集的 jdbcType 数据转换成java 数据类型。（至于具体的MyBatis是如何动态构建SQL语句的，我将准备专门一篇文章来讨论，敬请关注~）

##### SQL语句的执行以及封装查询结果集成List<E>
动态 SQL 语句生成之后，MyBatis 将执行 SQL 语句，并将可能返回的结果集转换成 List<E> 列表。MyBatis 在对结果集的处理中，支持结果集关系一对多和多对一的转换，并且有两种支持方式，一种为嵌套查询语句的查询，还有一种是嵌套结果集的查询。

#### 3.框架支撑层
##### 事务管理机制
事务管理机制对于 ORM 框架而言是不可缺少的一部分，事务管理机制的质量也是考量一个 ORM 框架是否优秀的一个标准，对于数据管理机制我已经在我的博文《深入理解mybatis原理》 MyBatis 事务管理机制 中有非常详细的讨论，感兴趣的读者可以点击查看。

##### 连接池管理机制
由于创建一个数据库连接所占用的资源比较大， 对于数据吞吐量大和访问量非常大的应用而言，连接池的设计就显得非常重要，对于连接池管理机制我已经在我的博文《深入理解mybatis原理》 Mybatis数据源与连接池 中有非常详细的讨论，感兴趣的读者可以点击查看。

##### 缓存机制
为了提高数据利用率和减小服务器和数据库的压力，MyBatis 会对于一些查询提供会话级别的数据缓存，会将对某一次查询，放置到 SqlSession 中，在允许的时间间隔内，对于完全相同的查询，MyBatis 会直接将缓存结果返回给用户，而不用再到数据库中查找。（至于具体的 MyBatis 缓存机制，我将准备专门一篇文章来讨论，敬请关注~）

##### SQL语句的配置方式
传统的 MyBatis 配置 SQL 语句方式就是使用XML文件进行配置的，但是这种方式不能很好地支持面向接口编程的理念，为了支持面向接口的编程，MyBatis 引入了 Mapper 接口的概念，面向接口的引入，对使用注解来配置 SQL 语句成为可能，用户只需要在接口上添加必要的注解即可，不用再去配置XML文件了，但是，目前的 MyBatis 只是对注解配置 SQL 语句提供了有限的支持，某些高级功能还是要依赖 XML 配置文件配置 SQL 语句。

#### 4.引导层
引导层是配置和启动 MyBatis 配置信息的方式。MyBatis 提供两种方式来引导 MyBatis ：基于XML配置文件的方式和基于Java API 的方式，读者可以参考我的另一片博文：Java Persistence with MyBatis 3(中文版) 第二章 引导MyBatis

### 二、MyBatis的主要构件及其相互关系
![image](http://img.blog.csdn.net/20141028140852531?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVhbmxvdWlz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

