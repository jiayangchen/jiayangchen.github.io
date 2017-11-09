---
layout: post
title: "聊一聊 Spring MVC"
subtitle: ""
date: 2017-07-31
author: "ChenJY"
header-img: "img/db.jpg"
catalog: true
tags: 
    - Spring
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>

### MVC框架是什么

模型-视图-控制器（MVC）是一个众所周知的以设计界面应用程序为基础的设计模式。它主要通过分离模型、视图及控制器在应用程序中的角色将业务逻辑从界面中解耦。通常，模型负责封装应用程序数据在视图层展示。视图仅仅只是展示这些数据，不包含任何业务逻辑。控制器负责接收来自用户的请求，并调用后台服务（manager或者dao）来处理业务逻辑。处理后，后台业务层可能会返回了一些数据在视图层展示。控制器收集这些数据及准备模型在视图层展示。MVC模式的核心思想是将业务逻辑从界面中分离出来，允许它们单独改变而不会相互影响。

![image](http://ww1.sinaimg.cn/mw690/6941baebtw1epg9am5105g20c809nt99.gif)

在Spring MVC应用程序中，模型通常由POJO对象组成，它在业务层中被处理，在持久层中被持久化。视图通常是用JSP标准标签库（JSTL）编写的JSP模板。控制器部分是由dispatcher servlet负责，在本教程中我们将会了解更多它的相关细节。
一些开发人员认为业务层和DAO层类是MVC模型组件的一部分。我对此持有不同的意见。我不认为业务层及DAO层类为MVC框架的一部分。通常一个web应用是3层架构，即数据-业务-表示。MVC实际上是表示层的一部分。

![image](http://ww1.sinaimg.cn/mw690/6941baebtw1epg9anj7l3j20cx09it9x.jpg)

Dispatcher Servlet(Spring控制器)

在最简单的Spring MVC应用程序中，控制器是唯一的你需要在Java web部署描述文件（即web.xml文件）中配置的Servlet。Spring MVC控制器 ——通常称作Dispatcher Servlet，实现了前端控制器设计模式。并且每个web请求必须通过它以便它能够管理整个请求的生命周期。

当一个web请求发送到Spring MVC应用程序，dispatcher servlet首先接收请求。然后它组织那些在Spring web应用程序上下文配置的（例如实际请求处理控制器和视图解析器）或者使用注解配置的组件，所有的这些都需要处理该请求。

![image](http://ww4.sinaimg.cn/mw690/6941baebtw1epg9al8bv6j20f90aqjrx.jpg)

在Spring3.0中定义一个控制器类，这个类必须标有@Controller注解。当有@Controller注解的控制器收到一个请求时，它会寻找一个合适的handler方法去处理这个请求。这就需要控制器通过一个或多个handler映射去把每个请求映射到handler方法。为了这样做，一个控制器类的方法需要被@RequestMapping注解装饰，使它们成为handler方法。

handler方法处理完请求后，它把控制权委托给视图名与handler方法返回值相同的视图。为了提供一个灵活的方法，一个handler方法的返回值并不代表一个视图的实现而是一个逻辑视图，即没有任何文件扩展名。你可以将这些逻辑视图映射到正确的实现，并将这些实现写入到上下文文件，这样你就可以轻松的更改视图层代码甚至不用修改请求handler类的代码。
为一个逻辑名称匹配正确的文件是视图解析器的责任。一旦控制器类已将一个视图名称解析到一个视图实现。它会根据视图实现的设计来渲染对应对象。

### MVC 框架的不足
1. 每次请求必须经过“控制器->模型->视图”这个流程，用户才能看到最终的展现的界面，这个过程似乎有些复杂。
2. 实际上视图是依赖于模型的，换句话说，如果没有模型，视图也无法呈现出最终的效果。
3. 渲染视图的过程是在服务端来完成的，最终呈现给浏览器的是带有模型的视图页面，性能无法得到很好的优化。

为了使数据展现过程更加直接，并且提供更好的用户体验，我们有必要对 MVC 模式进行改进。不妨这样来尝试，首先从浏览器发送 AJAX 请求，然后服务端接受该请求并返回 JSON 数据返回给浏览器，最后在浏览器中进行界面渲染。

改进后的 MVC 模式如下图所示：
![image](http://i.imgur.com/QnrL8i1.png)

也就是说，我们输入的是 AJAX 请求，输出的是 JSON 数据，市面上有这样的技术来实现这个功能吗？答案是 REST。

REST 全称是 Representational State Transfer（表述性状态转移），它是 Roy Fielding 博士在 2000 年写的一篇关于软件架构风格的论文，此文一出，威震四方！国内外许多知名互联网公司纷纷开始采用这种轻量级的 Web 服务，大家习惯将其称为 RESTful Web Services，或简称 REST 服务。

如果将浏览器这一端视为前端，而服务器那一端视为后端的话，可以将以上改进后的 MVC 模式简化为以下前后端分离模式：

![iamge](http://i.imgur.com/qu5dZn1.png)

可见，有了 REST 服务，前端关注界面展现，后端关注业务逻辑，分工明确，职责清晰。

### 参考资料
* [Spring MVC 入门示例讲解](http://www.importnew.com/15141.html)
* [从 MVC 到前后端分离](http://www.importnew.com/21589.html)

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 网络平台转载请联系 Chen.Jiayang@foxmail.com