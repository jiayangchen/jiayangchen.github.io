---
layout: post
title: "在 Github Pages 搭建的静态博客中添加 https 访问"
subtitle: ""
date: 2017-12-25
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 随笔
---

先看配置完成之后的效果，如下图：

![](http://o9oomuync.bkt.clouddn.com/addhttpsexample.png)

### Introduction

自从博客搭建伊始，就一直想配一个 `HTTPS` 上去，付费的自然用不起，况且博主也只是想讲究有个形式而已，裸的 `HTTP` 网站经常被 `Chrome` 提醒不安全，而且地址栏逼格不高，所以没事干就折腾一下。

### Tools

使用到的方法也非常简单，网上资料较多，但是部分教程步骤不全，因此准备整理一份齐全的，大家可以学一学，让自己的个人网站提高一下姿势水平。配置过程中用到了 `CloudFare` 和 `GoDaddy`，就这两样，对于个人域名不是在 `GoDaddy` 上申请的，无伤大雅，找到能设置你 `DNS` 服务器的页面就行了。

### Procedure

#### Sign up & Scan

第一步，先到 `CloudFare` 上申请一个账号，之后点击 `add Website` 输入你自己的域名，注意不要写 `www` 前缀，再点击扫描按钮，等待`半分钟`左右，当底部显示 `scan complete`，点击继续`设置`。

![](http://o9oomuync.bkt.clouddn.com/addhttpsprocedure1.png)

我们可以看到 `CNAME` 这个选项即是你在 `Github` 的 `CNAME` 文件中关联的 `username.github.io` 结尾的链接，点击 `continue` 继续。

![](http://o9oomuync.bkt.clouddn.com/addhttpsscanResult.png)

#### Pick up

没钱的我选择 `Free` 版，土豪请自便，顺便留下`微信`交个朋友？

![](http://o9oomuync.bkt.clouddn.com/addhttpsfreeWebsite.png)

#### CloudFare NameServer

之后便会出现 `CloudFare` 给出的域名服务器，保存好。

![](http://o9oomuync.bkt.clouddn.com/addhttpsnewNameServer.png)

#### GoDaddy DNS

转到 `GoDaddy` 上，登录自己的账号，通过输入自己的域名转到 `DNS` 管理页面。

![](http://o9oomuync.bkt.clouddn.com/addhttpssearchDNS.png)

点击修改

![](http://o9oomuync.bkt.clouddn.com/addhttpsdnsServer.png)

类型选择自定义，按照得到的 `CloudFare` 域名服务器进行填写，点击保存，等待大概 `5-10 min` 生效。

![](http://o9oomuync.bkt.clouddn.com/addhttpsmodifyDNS.png)

##### Overview

点击工具栏 `Overview` 按钮观察到 `Status` 变为 `Active` 之后，进入下一步。

![](http://o9oomuync.bkt.clouddn.com/addhttpsactive.png)

##### Crypto

点击工具栏 `Crypto` 按钮，设置 `Flexible SSL`。

![](http://o9oomuync.bkt.clouddn.com/addhttpsflexible.png)

##### Page Rules

最后，点击 `Page Rules`，点击 `create page rules` 添加页面重定向。

![](http://o9oomuync.bkt.clouddn.com/addhttpspagerule.png)

可以选择 `Always Use HTTPS`，并对你的域名使用通配符 `*`

![](http://o9oomuync.bkt.clouddn.com/addhttpslast.png)

### QA

按照本博客一贯的传统（问：哦？什么传统从什么时候开始的，我怎么不知道？ 答：现在开始），其中涉及的原理是一定要讲清楚的。那么我们下面来谈谈为什么这么做就能 `HTTPS` 了呢？和正规网站的 `HTTPS` 有什么不同呢？前者在博客 [HTTP & HTTPS, Session & Cookie](https://chenjiayang.me/2017/07/29/https-cookie-session/) 中有解答，感兴趣的可以看看，而后者的话，其实很简单。

其中原理有点像反向代理，还记得刚刚选择的 flexible SSL 么？其实它相当于每次你的请求先发到 CloudFare 的域名服务器，再由 CLoudFare 的域名服务器转发你的请求到实际资源服务器，取回结果之后再经 CLoudFare 返给用户。而其中，SSL 只加密用户至 CloudFlare 之间的通信，但CloudFlare 至实际资源服务器之间的通信依旧是明文的，所以其实全程通信来看还是不安全的。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="https://unsplash.com/@paramir?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ehud Neuhaus"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Ehud Neuhaus</span></a>





