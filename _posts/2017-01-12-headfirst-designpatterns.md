---
layout: post
title: "设计模式一 —— 单例模式"
subtitle: ""
date: 2017-01-12
author: "ChenJY"
header-img: "img/design_patterns.jpg"
catalog: true
tags: 
    - 设计模式
---

### 概念：
Java中单例模式是一种常见的设计模式，单例模式的写法有好几种，主要的包括三种：懒汉式、饿汉式、登记式、枚举式。
单例模式有以下特点：
* 1、单例类只能有一个实例。
* 2、单例类必须自己创建自己的唯一实例。
* 3、单例类必须给所有其他对象提供这一实例。

### 优点：
* 1、在内存中只有一个对象，节省内存空间。
* 2、避免频繁的创建销毁对象，可以提高性能。
* 3、避免对共享资源的多重占用。
* 4、可以全局访问。

### 懒汉式单例
```java
//懒汉式单例类.在第一次调用的时候实例化自己   
public class Singleton {  
    private Singleton() {}  
    private static Singleton single=null;  
    //静态工厂方法   
    public static Singleton getInstance() {  
         if (single == null) {    
             single = new Singleton();  
         }    
        return single;  
    }  
}
```

<b>Singleton</b> 通过将构造方法限定为 <b>private</b> 避免了类在外部被实例化，在同一个虚拟机范围内，<b>Singleton</b> 的唯一实例只能通过 getInstance() 方法访问。

但是以上懒汉式单例的实现没有考虑线程安全问题，它是线程不安全的，并发环境下很可能出现多个Singleton实例，如果现在存在着线程A和B，代码执行情况是这个样子的，线程A执行到了 <b>If(singleton == null)</b> ，线程B执行到了 <b>Singleton = new Singleton()</b> ;线程B虽然实例化了一个 Singleton，但是对于线程A来说判断singleton还是未初始化的，所以线程A还会对singleton进行初始化。

#### 第一次尝试
我们可以通过添加 <b>Synchronized</b> 关键词来解决同步问题：
```java
public class Singleton {  
    private Singleton() {}  
    private static Singleton single=null;  
    //静态工厂方法   
    public static Singleton getInstance() {  
         if (single == null) {    
             synchronized(Singleton.class){
                single = new Singleton();  
             }
         }    
        return single;  
    }  
}
```
通过 __Synchronized__ 关键词我们貌似成功的解决了多线程的问题，但是仔细想想还是不正确，因为A，B两个线程可能同时等待进入<b>synchronized(Singleton.class)</b> 这句话，而之后又不再判断一次single是否为null，那么还是会出现多个实例（因为AB均可以先后进入这个代码块）。因此，我们更好的解决办法是双重判断：      

#### 第二次尝试
```java
public class Singleton {  
    private Singleton() {}  
    private static Singleton single=null;  
    //静态工厂方法   
    public static Singleton getInstance() {  
         if (single == null) {    
             synchronized(Singleton.class){
                 if(single == null){
                    single = new Singleton();  
                 }
             }
         }    
        return single;  
    }  
}
```
当然，我们也可以将 <b>Synchronized</b> 关键词放在public方法上，同样可以起到效果，只是锁定这个方法会带来较大的效率损失。

#### 改进方法
```java
//静态内部类，既实现了线程安全，又避免了同步带来的性能影响
public class Singleton {    
    private static class LazyHolder {    
       private static final Singleton INSTANCE = new Singleton();    
    }    
    private Singleton (){}    
    public static final Singleton getInstance() {    
       return LazyHolder.INSTANCE;    
    }    
}  
```

```java
//枚举单例实现
public class Singleton {
    private static enum Singleton {
        INSTANCE;
        private Singleton single;
        private Singleton(){
            single = new Singleton();
        }
        private Singleton getInstance(){
            return single;
        }
    }
}
```

### 饿汉式单例
```java
//饿汉式单例先自己new一个实例，在你需要的时候直接返给你
public class Singleton{
       private static final Singleton singleton = new Singleton();
       private Singleton(){}
       public static Singleton getInstance(){
          return singleton;
       }
}
```
* 优点是：写起来比较简单，而且不存在多线程同步问题，避免了 <b>Synchronized</b> 所造成的性能问题；
* 缺点是：当类 <b>SingletonTest</b> 被加载的时候，会初始化 static 的 instance，静态变量被创建并分配内存空间，从这以后，这个static的instance对象便一直占着这段内存（即便你还没有用到这个实例），当类被卸载时，静态变量被摧毁，并释放所占有的内存，因此在某些特定条件下会耗费内存。

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自 <a href="https://unsplash.com/" target="_blank"><b> unsplash </b></a>