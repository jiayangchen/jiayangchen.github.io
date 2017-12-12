---
layout: post
title: "【转载】Java 递归、非递归遍历二叉树"
subtitle: ""
date: 2017-08-02
author: "hairongtian"
header-img: "img/db.jpg"
catalog: true
tags: 
    - 数据结构
    - 转载
---

> [原文链接](http://blog.csdn.net/hairongtian/article/details/7930937)，作者：hairongtian

```java
import java.util.Stack;    
import java.util.HashMap;    
    
public class BinTree {    
    private char date;    
    private BinTree lchild;    
    private BinTree rchild;    
    
    public BinTree(char c) {    
        date = c;    
    }    
    
    // 先序遍历递归     
    public static void preOrder(BinTree t) {    
        if (t == null) {    
            return;    
        }    
        System.out.print(t.date);    
        preOrder(t.lchild);    
        preOrder(t.rchild);    
    }    
    
    // 中序遍历递归     
    public static void InOrder(BinTree t) {    
        if (t == null) {    
            return;    
        }    
        InOrder(t.lchild);    
        System.out.print(t.date);    
        InOrder(t.rchild);    
    }    
    
    // 后序遍历递归     
    public static void PostOrder(BinTree t) {    
        if (t == null) {    
            return;    
        }    
        PostOrder(t.lchild);    
        PostOrder(t.rchild);    
        System.out.print(t.date);    
    }    
    
    // 先序遍历非递归     
    public static void preOrder2(BinTree t) {    
        Stack<BinTree> s = new Stack<BinTree>();    
        while (t != null || !s.empty()) {    
            while (t != null) {    
                System.out.print(t.date);    
                s.push(t);    
                t = t.lchild;    
            }    
            if (!s.empty()) {    
                t = s.pop();    
                t = t.rchild;    
            }    
        }    
    }    
    
    // 中序遍历非递归     
    public static void InOrder2(BinTree t) {    
        Stack<BinTree> s = new Stack<BinTree>();    
        while (t != null || !s.empty()) {    
            while (t != null) {    
                s.push(t);    
                t = t.lchild;    
            }    
            if (!s.empty()) {    
                t = s.pop();    
                System.out.print(t.date);    
                t = t.rchild;    
            }    
        }    
    }    
    
    // 后序遍历非递归     
    public static void PostOrder2(BinTree t) {    
        Stack<BinTree> s = new Stack<BinTree>();    
        Stack<Integer> s2 = new Stack<Integer>();    
        Integer i = new Integer(1);    
        while (t != null || !s.empty()) {    
            while (t != null) {    
                s.push(t);    
                s2.push(new Integer(0));    
                t = t.lchild;    
            }    
            while (!s.empty() && s2.peek().equals(i)) {    
                s2.pop();    
                System.out.print(s.pop().date);    
            }    
    
            if (!s.empty()) {    
                s2.pop();    
                s2.push(new Integer(1));    
                t = s.peek();    
                t = t.rchild;    
            }    
        }    
    }    
    
    public static void main(String[] args) {    
        BinTree b1 = new BinTree('a');    
        BinTree b2 = new BinTree('b');    
        BinTree b3 = new BinTree('c');    
        BinTree b4 = new BinTree('d');    
        BinTree b5 = new BinTree('e');    
    
        /**  
         *      a   
         *     / /  
         *    b   c  
         *   / /  
         *  d   e  
         */    
        b1.lchild = b2;    
        b1.rchild = b3;    
        b2.lchild = b4;    
        b2.rchild = b5;    
    
        BinTree.preOrder(b1);    
        System.out.println();    
        BinTree.preOrder2(b1);    
        System.out.println();    
        BinTree.InOrder(b1);    
        System.out.println();    
        BinTree.InOrder2(b1);    
        System.out.println();    
        BinTree.PostOrder(b1);    
        System.out.println();    
        BinTree.PostOrder2(b1);    
    }    
}
```

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>