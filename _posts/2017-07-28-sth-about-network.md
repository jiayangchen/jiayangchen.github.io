---
layout: post
title: "计算机网络 —— 网络分层架构"
subtitle: ""
date: 2017-07-28
author: "ChenJY"
header-img: "img/db.jpg"
catalog: true
tags: 
    - 网络
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>

因为本身学院的课非常“国际化”，基本上将软工和CS常见的一些课，例如编译原理、汇编语言、计算机网络等都用 CMU 的 ICS 和 MIT 的 CSE 代替了，因此前面那些课我都没学过，编译汇编两个还好，对实际开发编码影响没那么大，但是计网从自己的实际需求出发，感觉掌握的还是太浅了，而且从找实习的情况来看，外面的面试官又不知道你学没学过，都是默认计网是必修课，答的不好就会刷掉你，因此还是写一篇总结，完善一下计网里的基础知识吧。

### 网络分层架构
![image](https://gss0.baidu.com/-vo3dSag_xI4khGko9WTAnF6hhy/zhidao/wh%3D600%2C800/sign=ac73628c32c79f3d8fb4ec368a91e129/203fb80e7bec54e7539f3d6db0389b504fc26a55.jpg)

#### OSI分层
总共七层，由上而下为：
* 应用层：直接为<b>用户的应用进程提供服务</b>，其中应用层协议包括万维网的<b>HTTP协议，电子邮件的SMTP协议，文件传输的FTP协议等</b>。
* 表示层：它对来自应用层的命令和数据进行解释，对各种语法赋予相应的含义，并按照一定的格式传送给会话层。其主要功能是<b>“处理用户信息的表示问题，如编码、数据格式转换和加密解密”等</b>。
* 会话层：是用户应用程序和网络之间的接口，主要任务是：向两个实体的表示层提供建立和使用连接的方法。将不同实体之间的表示层的连接称为会话。因此会话层的任务就是<b>组织和协调两个会话进程之间的通信，并对数据交换进行管理</b>。
* 运输层：传输层提供会话层和网络层之间的传输服务，这种服务从会话层获得数据，并在必要时，对数据进行分割。然后，传输层将数据传递到网络层，并确保数据能正确无误地传送到网络层。因此，传输层<b>负责提供两节点之间数据的可靠传送，当两节点的联系确定之后，传输层则负责监督工作</b>。
* 网络层：其主要任务是：<b>通过路由选择算法，为报文或分组通过通信子网选择最适当的路径</b>。该层控制数据链路层与传输层之间的信息转发，建立、维持和终止网络的连接。
* 数据链路层：该层的主要功能是：<b>通过各种控制协议，将有差错的物理信道变为无差错的、能可靠传输数据帧的数据链路</b>。
* 物理层：物理层的主要功能是：<b>利用传输介质为数据链路层提供物理连接，实现比特流的透明传输</b>。

#### TCP/IP 的体系结构
四层，由上而下为：
* 网络接口层：定义像地址解析协议(Address Resolution Protocol,ARP)这样的协议，提供TCP/IP协议的数据结构和实际物理硬件之间的接口。
* 网际层：本层包含IP协议、RIP协议(Routing Information Protocol，路由信息协议)，负责数据的包装、寻址和路由。同时还包含网间控制报文协议(Internet Control Message Protocol,ICMP)用来提供网络诊断信息。
* 运输层TCP/UDP：提供TCP和UDP
* 应用层：对应于OSI七层参考模型的应用层和表达层

#### 五层协议的体系结构
在学习计算机网络原理是往往采用折中的办法，即综合OSI和TCP/IP的优点，采用一种只有五层协议的体系结构，这样既简洁又能将概念阐述清楚，由上而下为：
应用层,运输层,网络层,数据链路层,物理层

### ARP协议工作原理
首先，每台主机都会在自己的ARP缓冲区中建立一个 ARP列表，以表示IP地址和MAC地址的对应关系。当源主机需要将一个数据包要发送到目的主机时，会首先检查自己 ARP列表中是否存在该 IP地址对应的MAC地址，如果有，就直接将数据包发送到这个MAC地址；如果没有，就向本地网段发起一个ARP请求的广播包，查询此目的主机对应的MAC地址。此ARP请求数据包里包括源主机的IP地址、硬件地址、以及目的主机的IP地址。网络中所有的主机收到这个ARP请求后，会检查数据包中的目的IP是否和自己的IP地址一致。如果不相同就忽略此数据包；如果相同，该主机首先将发送端的MAC地址和IP地址添加到自己的ARP列表中，如果ARP表中已经存在该IP的信息，则将其覆盖，然后给源主机发送一个 ARP响应数据包，告诉对方自己是它需要查找的MAC地址；源主机收到这个ARP响应数据包后，将得到的目的主机的IP地址和MAC地址添加到自己的ARP列表中，并利用此信息开始数据的传输。如果源主机一直没有收到ARP响应数据包，表示ARP查询失败。

### 在浏览器中输入www.baidu.com后执行的全部过程
#### 点击网址后，应用层的DNS协议会将网址解析为IP地址；
DNS查找过程：
1. 浏览器会检查缓存中有没有这个域名对应的解析过的IP地址，如果缓存中有，这个解析过程就将结束。
2. 如果用户的浏览器缓存中没有，浏览器会查找操作系统缓存（hosts文件）中是否有这个域名对应的DNS解析结果。
3. 若还没有，此时会发送一个数据包给DNS服务器，DNS服务器找到后将解析所得IP地址返回给用户。

#### 在应用层，浏览器会给web服务器发送一个HTTP请求；
请求头为：GET http://www.baidu.com/HTTP/1.1

#### 在传输层，（上层的传输数据流分段）HTTP数据包会嵌入在TCP报文段中；
TCP报文段需要设置端口，接收方（百度）的HTTP端口默认是80，本机的端口是一个1024-65535之间的随机整数，这里假设为1025，这样TCP报文段由TCP首部（包含发送方和接收方的端口信息）+ HTTP数据包组成。

#### 在网络层中，TCP报文段再嵌入IP数据包中；
IP数据包需要知道双方的IP地址，本机IP地址假定为192.168.1.5，接受方IP地址为220.181.111.147（百度），这样IP数据包由IP头部（IP地址信息）+TCP报文段组成。

#### 在网络接口层，IP数据包嵌入到数据帧（以太网数据包）中在网络上传送；
数据帧中包含源MAC地址和目的MAC地址（通过ARP地址解析协议得到的）。这样数据帧由头部（MAC地址）+IP数据包组成。

#### 数据包经过多个网关的转发到达百度服务器，请求对应端口的服务；
服务接收到发送过来的以太网数据包开始解析请求信息，从以太网数据包中提取IP数据包—>TCP报文段—>HTTP数据包，并组装为有效数据交与对应线程池中分配的线程进行处理，在这个过程中，生成相应request、response对象。

#### 请求处理完成之后，服务器发回一个HTTP响应；
请求处理程序会阅读请求及它的参数和cookies。它会读取也可能更新一些数据，并将数据存储在服务器上。处理完毕后，数据通过response对象给客户输出信息，输出信息也需要拼接HTTP协议头部分，关闭后断开连接。断开后，服务器端自动注销request、response对象，并将释放对应线程的使用标识（一般一个请求单独由一个线程处理，部分特殊情况有一个线程处理多个请求的情况）。
响应头为：HTTP/1.1200 OK

#### 浏览器以同样的过程读取到HTTP响应的内容（HTTP响应数据包），然后浏览器对接收到的HTML页面进行解析，把网页显示出来呈现给用户。
客户端接收到返回数据，去掉对应头信息，形成也可以被浏览器认识的页面HTML字符串信息，交与浏览器翻译为对应页面规则信息展示为页面内容。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 网络平台转载请联系 Chen.Jiayang@foxmail.com
