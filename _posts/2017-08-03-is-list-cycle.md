---
layout: post
title: "Java 判断单链表是否有环"
subtitle: "三种方法"
date: 2017-08-03
author: "ChenJY"
header-img: "img/db.jpg"
catalog: true
tags: 
    - 算法
---

### 方法一：

首先从头节点开始，依次遍历单链表的每一个节点。每遍历到一个新节点，就从头节点重新遍历新节点之前的所有节点，用新节点ID和此节点之前所有节点ID依次作比较。如果发现新节点之前的所有节点当中存在相同节点ID，则说明该节点被遍历过两次，链表有环；如果之前的所有节点当中不存在相同的节点，就继续遍历下一个新节点，继续重复刚才的操作。

例如这样的链表：A->B->C->D->B->C->D， 当遍历到节点D的时候，我们需要比较的是之前的节点A、B、C，不存在相同节点。这时候要遍历的下一个新节点是B，B之前的节点A、B、C、D中恰好也存在B，因此B出现了两次，判断出链表有环。

假设从链表头节点到入环点的距离是D，链表的环长是S。D+S是链表总节点数。那么算法的时间复杂度是0+1+2+3+….+(D+S-1) = (D+S-1)*(D+S)/2 ， 可以简单地理解成 O(N*N)。而此算法没有创建额外存储空间，空间复杂度可以简单地理解成为O(1)。

### 方法二：

首先创建一个以节点ID为键的HashSet集合，用来存储曾经遍历过的节点。然后同样是从头节点开始，依次遍历单链表的每一个节点。每遍历到一个新节点，就用新节点和HashSet集合当中存储的节点作比较，如果发现HashSet当中存在相同节点ID，则说明链表有环，如果HashSet当中不存在相同的节点ID，就把这个新节点ID存入HashSet，之后进入下一节点，继续重复刚才的操作。

这个方法在流程上和方法一类似，本质的区别是使用了HashSet作为额外的缓存。

假设从链表头节点到入环点的距离是D，链表的环长是S。而每一次HashSet查找元素的时间复杂度是O(1), 所以总体的时间复杂度是1*(D+S)=D+S，可以简单理解为O(N)。而算法的空间复杂度还是D+S-1，可以简单地理解成O(N)。

### 方法三：

首先创建两个指针1和2（在Java里就是两个对象引用），同时指向这个链表的头节点。然后开始一个大循环，在循环体中，让指针1每次向下移动一个节点，让指针2每次向下移动两个节点，然后比较两个指针指向的节点是否相同。如果相同，则判断出链表有环，如果不同，则继续下一次循环。

例如链表A->B->C->D->B->C->D，两个指针最初都指向节点A，进入第一轮循环，指针1移动到了节点B，指针2移动到了C。第二轮循环，指针1移动到了节点C，指针2移动到了节点B。第三轮循环，指针1移动到了节点D，指针2移动到了节点D，此时两指针指向同一节点，判断出链表有环。

此方法也可以用一个更生动的例子来形容：在一个环形跑道上，两个运动员在同一地点起跑，一个运动员速度快，一个运动员速度慢。当两人跑了一段时间，速度快的运动员必然会从速度慢的运动员身后再次追上并超过，原因很简单，因为跑道是环形的。

```java
import java.util.HashMap;

import demo6.LinkReverse2.Node;


/**
 * 判断链表是否有环的方法
 * @author mengfeiyang
 *
 */
public class LinkLoop {

    public static boolean hasLoop(Node n){
        //定义两个指针tmp1,tmp2
        Node tmp1 = n;
        Node tmp2 = n.next;

        while(tmp2!=null){

            int d1 = tmp1.val;
            int d2 = tmp2.val;
            if(d1 == d2)return true;//当两个指针重逢时，说明存在环，否则不存在。
            tmp1 = tmp1.next;  //每次迭代时，指针1走一步，指针2走两步
            tmp2 = tmp2.next.next;
            if(tmp2 == null)return false;//不存在环时，退出

        }
        return true; //如果tmp2为null，说明元素只有一个，也可以说明是存在环
    }

    //方法2：将每次走过的节点保存到hash表中，如果节点在hash表中，则表示存在环
    public static boolean hasLoop2(Node n){
        Node temp1 = n;
        HashMap<Node,Node> ns = new HashMap<Node,Node>();
        while(n!=null){
            if(ns.get(temp1)!=null)return true;
            else ns.put(temp1, temp1);
            temp1 = temp1.next;
            if(temp1 == null)return false;
        }
        return true;
    }

    public static void main(String[] args) {
        Node n1 = new Node(1);
        Node n2 = new Node(2);
        Node n3 = new Node(3);
        Node n4 = new Node(4);
        Node n5 = new Node(5);

        n1.next = n2;
        n2.next = n3;
        n3.next = n4;
        n4.next = n5;
        n5.next = n1;  //构造一个带环的链表,去除此句表示不带环

        System.out.println(hasLoop(n1));
        System.out.println(hasLoop2(n1));
    }
}
```

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 网络平台转载请联系 Chen.Jiayang@foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>