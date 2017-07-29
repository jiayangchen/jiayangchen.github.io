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
1. 客户端浏览器通过DNS解析到www.baidu.com的IP地址220.181.27.48，通过这个IP地址找到客户端到服务器的路径。客户端浏览器发起一个HTTP会话到220.161.27.48，然后通过TCP进行封装数据包，输入到网络层。

2. 客户端的传输层，把HTTP会话请求分成报文段，添加源和目的端口，如服务器使用80端口监听客户端的请求，客户端由系统随机选择一个端口如5000，与服务器进行交换，服务器把相应的请求返回给客户端的5000端口。然后使用IP层的IP地址查找目的端。

3. 客户端的网络层不用关系应用层或者传输层的东西，主要做的是通过查找路由表确定如何到达服务器，期间可能经过多个路由器，这些都是由路由器来完成的工作，我不作过多的描述，无非就是通过查找路由表决定通过那个路径到达服务器。

4. 客户端的链路层，包通过链路层发送到路由器，通过邻居协议查找给定IP地址的MAC地址，然后发送ARP请求查找目的地址，如果得到回应后就可以使用ARP的请求应答交换的IP数据包现在就可以传输了，然后发送IP数据包到达服务器的地址。

