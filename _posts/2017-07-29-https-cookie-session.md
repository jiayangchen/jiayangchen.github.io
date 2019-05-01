---
layout: post
title: "HTTP & HTTPS, Session & Cookie"
subtitle: ""
date: 2017-07-29
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 计算机网络
---

### HTTP 和 HTTPS 
#### 基本概念
1. HTTP 是互联网上应用最为广泛的一种网络协议，是一个客户端和服务器端请求和应答的标准（TCP），用于从WWW服务器传输超文本到本地浏览器的传输协议，它可以使浏览器更加高效，使网络传输减少。

2. HTTPS 是以安全为目标的 HTTP 通道，简单讲是 HTTP 的安全版，<b>即 HTTP 下加入SSL层</b>，HTTPS 的安全基础是 SSL，因此加密的详细内容就需要 SSL。

3. HTTPS 协议的主要作用可以分为两种：一种是<b>建立一个信息安全通道，来保证数据传输的安全；另一种就是确认网站的真实性</b>。

#### HTTP 协议之响应
状态代码有三位数字组成，第一个数字定义了响应的类别，且有五种可能取值：
* 1xx：<b>指示信息</b> -- 表示请求已接收，继续处理
* 2xx：<b>成功</b> -- 表示请求已被成功接收、理解、接受
* 3xx：<b>重定向</b> -- 要完成请求必须进行更进一步的操作
* 4xx：<b>客户端错误</b> -- 请求有语法错误或请求无法实现
* 5xx：<b>服务器端错误</b> -- 服务器未能实现合法的请求

常见状态代码、状态描述、说明：
* 200 OK      //客户端请求成功
* 400 Bad Request  //客户端请求有语法错误，不能被服务器所理解
* 401 Unauthorized //请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
* 403 Forbidden  //服务器收到请求，但是拒绝提供服务
* 404 Not Found  //请求资源不存在，eg：输入了错误的URL
* 500 Internal Server Error //服务器发生不可预期的错误
* 503 Server Unavailable  //服务器当前不能处理客户端的请求，一段时间后可能恢复正常

#### HTTPS 工作原理

1. 客户使用 https 的 URL 访问 Web 服务器，要求与 Web 服务器建立 SSL 连接，连接到 Server 的 443 端口。

2. 采用 HTTPS 的服务器必须有一套<b>数字证书</b>，既可以自己制作也可以向组织申请，区别就是自己颁发的证书需要客户端验证通过，才可以继续访问，而使用受信任的公司申请的证书则不会弹出提示页面。这套证书其实就是<b>一对公钥和私钥</b>，可以类比成锁头和锁钥，可以把锁头给别人锁住东西，然后只有你自己能开锁。Web 服务器收到客户端请求后，会将网站的证书信息（证书中包含公钥，证书的颁发机构，过期时间等）传送一份给客户端。

3. 客户端由 TLS 解析证书，先<b>验证公钥是否有效</b>，如发现异常弹出警告框提示证书存在问题，紧接着客户端的浏览器与 Web 服务器开始协商 SSL 连接的安全等级，也就是信息加密的等级。

4. 客户端的浏览器根据双方同意的安全等级，并且证书经验证不存在问题的话，就会建立<b>会话密钥（其实就是一个随机值）</b>，然后利用网站的<b>公钥将会话密钥加密，并传送给网站</b>。

5. Web 服务器利用自己的<b>私钥解密出会话密钥</b>。这样客户端和服务器端就都有了这个密钥，就可以用它来加密信息了。采用<b>对称加密</b>的方法，<b>将信息和私钥的一部分通过某种算法混合到一起</b>，这样除非知道私钥，否则无法获取内容，而正好客户端与服务器端都知道这个私钥。

6. Web 服务器利用会话密钥加密与客户端之间的通信。

### Session 和 Cookie
> [原文链接](https://www.zhihu.com/question/19786827)，来自知乎,作者：[轩辕志远](https://www.zhihu.com/people/xuanyuanzhiyuan/answers)

1. 由于 HTTP 协议是<b>无状态的协议</b>，所以服务端需要记录用户的状态时，就需要用某种机制<b>来识具体的用户</b>，这个机制就是 Session.典型的场景比如购物车，当你点击下单按钮时，由于 HTTP 协议无状态，所以并不知道是哪个用户操作的，所以服务端要为特定的用户创建了特定的 Session，用用于标识这个用户，并且跟踪用户，这样才知道购物车里面有几本书。这个 Session 是<b>保存在服务端的，有一个唯一标识</b>。在服务端保存 Session 的方法很多，内存、数据库、文件都有。集群的时候也要考虑 Session 的转移，在大型的网站，一般会有专门的Session服务器集群，用来保存用户会话，这个时候 Session 信息都是放在内存的，使用一些缓存服务比如 Memcached 之类的来放 Session。

2. 思考一下<b>服务端如何识别特定的客户？</b>这个时候 Cookie 就登场了。每次 HTTP 请求的时候，<b>客户端都会发送相应的 Cookie 信息到服务端</b>。实际上大多数的应用都是用 Cookie 来实现 Session 跟踪的，第一次创建 Session 的时候，服务端会在 HTTP 协议中告诉客户端，需要在 Cookie 里面记录一个 Session ID，以后每次请求把这个会话 ID 发送到服务器，我就知道你是谁了。有人问，如果客户端的浏览器禁用了  Cookie 怎么办？一般这种情况下，会使用一种叫做 URL 重写的技术来进行会话跟踪，即每次 HTTP 交互，URL后面都会被附加上一个诸如 sid=xxxxx 这样的参数，服务端据此来识别用户。

3. Cookie 其实还可以用在一些方便用户的场景下，设想你某次登陆过一个网站，下次登录的时候不想再次输入账号了，怎么办？这个信息可以写到 Cookie 里面，访问网站的时候，网站页面的脚本可以读取这个信息，就自动帮你把用户名给填了，能够方便一下用户。这也是 Cookie 名称的由来，给用户的一点甜头。

所以，总结一下：Session 是在服务端保存的一个数据结构，<b>用来跟踪用户的状态</b>，这个数据可以保存在集群、数据库、文件中；Cookie 是客户端保存用户信息的一种机制，用来<b>记录用户的一些信息</b>，也是实现 Session 的一种方式。

### 粘滞会话(Sticky Sessions)
当我们使用反向代理做负载均衡时，用户对同一内容的多次请求，可能被转发到了不同的后端服务器，若有3台服务器进行集群，用户发出一请求被分配至服务器A，保存了一些信息在session中，该用户再次发送请求被分配到服务器B，要用之前保存的信息，若服务器A和B之间没有session粘滞，那么服务器B就拿不到之前的信息，这样会导致一些问题。

怎么解决呢？从表面上看，我们需要做的就是调整高度策略，让用户在一次会话周期内的所有请求始终转发到一台特定的后端服务器上，这种机制也称为粘滞会话（Sticky Sessions）,要实现它的关键在于如何设计持续性调度算法。既然要让高度器可以识别用户，那么将用户的 I P地址作为识别标志最为合适，一些反向代理服务器对此都有支持，比如 Nginx。对于 Nginx,只需要在 upstream 中声明 ip_hash 即可，每个请求都访问 IP 的 hash 结果分配，这样每个访客固定访问一个后端服务器。

### 301,302状态码
[301,302状态码](http://blog.csdn.net/grandPang/article/details/47448395)

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>


