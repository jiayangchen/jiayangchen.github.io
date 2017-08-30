---
layout: post
title: "浅谈 Java 反射原理和注解工作原理"
subtitle: ""
date: 2017-08-30
author: "Jakob Jenkov"
header-img: "img/db.jpg"
catalog: true
tags: 
    - 面试
---

<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>

Java反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为Java语言的反射机制。

```java
/** 
 * 反射机制获取类有三种方法 
 */  
@Test  
public void testGetClass() throws ClassNotFoundException {  
    Class clazz = null;  
  
    //1 直接通过类名.Class的方式得到  
    clazz = Person.class;  
    System.out.println("通过类名: " + clazz);  
  
    //2 通过对象的getClass()方法获取,这个使用的少（一般是传的是Object，不知道是什么类型的时候才用）  
    Object obj = new Person();  
    clazz = obj.getClass();  
    System.out.println("通过getClass(): " + clazz);  
  
    //3 通过全类名获取，用的比较多，但可能抛出ClassNotFoundException异常  
    clazz = Class.forName("com.java.reflection.Person");  
    System.out.println("通过全类名获取: " + clazz);  
}  
```

### newInstance

利用newInstance创建对象：调用的类必须有无参的构造器

```java
/** 
 * Class类的newInstance()方法，创建类的一个对象。 
 */  
@Test  
public void testNewInstance()  
        throws ClassNotFoundException, IllegalAccessException, InstantiationException {  
  
    Class clazz = Class.forName("com.java.reflection.Person");  
  
    //使用Class类的newInstance()方法创建类的一个对象  
    //实际调用的类的那个 无参数的构造器（这就是为什么写的类的时候，要写一个无参数的构造器，就是给反射用的）  
    //一般的，一个类若声明了带参数的构造器，也要声明一个无参数的构造器  
    Object obj = clazz.newInstance();  
    System.out.println(obj);  
} 
```

### ClassLoader类加载器

![image](http://img.blog.csdn.net/20130625103818562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveGlhb3hpYW44MDIz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 注解

* 从 JDK5.0 开始,Java 增加了对元数据(MetaData)的支持,也就是Annotation(注释)
* Annotation其实就是代码里的特殊标记,这些标记可以在编译,类加载, 运行时被读取,并执行相应的处理.通过使用Annotation,程序员可以在不改变原有逻辑的情况下,在源文件中嵌入一些补充信息.
* Annotation 可以像修饰符一样被使用,可用于修饰包,类,构造器, 方法,成员变量, 参数,局部变量的声明,这些信息被保存在Annotation的 “name=value”对中.
* Annotation能被用来为程序元素(类,方法,成员变量等)设置元数据

#### 基本的 Annotation

使用 Annotation时要在其前面增加@符号,并把该Annotation 当成一个修饰符使用.用于修饰它支持的程序元素
三个基本的Annotation:
1. @Override:表示当前的方法定义将覆盖超类中的方法。
2. @Deprecated:用于表示某个程序元素(类,方法等)已过时
3. @SuppressWarnings:抑制编译器警告

使用注解最主要的部分在于对注解的处理，那么就会涉及到注解处理器。从原理上讲，注解处理器就是通过反射机制获取被检查方法上的注解信息，然后根据注解元素的值进行特定的处理。

#### 自定义 Annotation

定义新的 Annotation 类型使用 @interface 关键字，Annotation 的成员变量在 Annotation 定义中以无参数方法的形式来声明.其方法名和返回值定义了该成员的名字和类型.可以在定义 Annotation 的成员变量时为其指定初始值,指定成员变量的初始值可使用 default 关键字，没有成员定义的 Annotation 称为标记;包含成员变量的 Annotation 称为元数据 Annotation。

> [深入理解Java 注解原理](http://blog.csdn.net/zhang0558/article/details/52643016)，作者：红叶幽香
 