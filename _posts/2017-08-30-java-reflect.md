---
layout: post
title: "Java 注解工作原理"
subtitle: ""
date: 2017-08-30
author: "ChenJY"
header-img: "img/db.jpg"
catalog: true
tags: 
    - Java Tech
---

### 什么是注解
注解也叫 `元数据`，为方便在代码中添加信息提供的一种形式化方法。它们可以提供用来完整描述程序所需的信息，而这些信息无法用 Java 来表达，因此注解使得我们能够以将由编译器来测试和验证的格式，存储有关程序的额外信息。注解同样还有利于代码阅读和编译器类型检查等。Java SE5 内置了三种，定义在 `java.lang` 中的注解：

1. `@Override`：表示当前的方法定义将覆盖超类中的方法，如果方法签名对应不正确，编译器会发出错误提示。
2. `@Deprecated`：如果使用注解为它的元素，编译器会发出错误提示。
3. `@SuppressWarnings`：抑制编译器警告，或者是关闭不当的编译器警告信息。

Java 还提供另外四种注解用于创建新注解。

### 定义注解
定义注解时，会需要一些元注解（meta-annotation）的帮助，例如 `@Target` 和 `@Retention`。 @Target 用来定义你的主解将被用在什么地方，@Retention 定义注解在哪个级别可用，在源代码（SOURCE）中、类文件（CLASS） 亦或者运行时（RUNTIME）：

```java
@Target(ElementType.METHOD)  //指作用在一个方法上
@Retention(RetentionPolicy.RUNTIME) //指作用在运行时
public @interface Test {}
```

### 写一个自定义注解
创建自定义注解和创建一个接口相似，但是注解的 `interface` 关键字需要以 `@` 符号开头。

#### 自定义注解
```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Target(ElementType.METHOD)
@Inherited
@Retention(RetentionPolicy.RUNTIME)
    public @interface MethodInfo{
    String author() default 'Pankaj';
    String date();
    int revision() default 1;
    String comments();
}
```

#### 使用自定义注解

```java
import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class AnnotationExample {

public static void main(String[] args) {

    @Override
    @MethodInfo(author = 'Pankaj', comments = 'Main method', date = 'Nov 17 2012', revision = 1)
    public String toString() {
        return 'Overriden toString method';
    }

    @Deprecated
    @MethodInfo(comments = 'deprecated method', date = 'Nov 17 2012')
    public static void oldMethod() {
        System.out.println("old method, don't use it.");
    }

    @SuppressWarnings({ 'unchecked', 'deprecation' })
    @MethodInfo(author = 'Pankaj', comments = 'Main method', date = 'Nov 17 2012', revision = 10)
    public static void genericsTest() throws FileNotFoundException {
        List l = new ArrayList();
        l.add('abc');
        oldMethod();
    }
}
```

#### 利用反射解析自定义注解
注解的 `RetentionPolicy` 应该设置为 `RUNTIME` 否则 java 类的注解信息在执行过程中将不可用，那么我们也不能从中得到任何和注解有关的数据。

```java
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

public class AnnotationParsing {

public static void main(String[] args) {
    try {
    for (Method method : AnnotationParsing.class
        .getClassLoader()
        .loadClass(('com.journaldev.annotations.AnnotationExample'))
        .getMethods()) {
        // checks if MethodInfo annotation is present for the method
        if (method.isAnnotationPresent(com.journaldev.annotations.MethodInfo.class)) {
            try {
        // iterates all the annotations available in the method
                for (Annotation anno : method.getDeclaredAnnotations()) {
                    System.out.println('Annotation in Method ''+ method + '' : ' + anno);
                    }
                MethodInfo methodAnno = method.getAnnotation(MethodInfo.class);
                if (methodAnno.revision() == 1) {
                    System.out.println('Method with revision no 1 = '+ method);
                    }

            } catch (Throwable ex) {
                    ex.printStackTrace();
                    }
        }
    }
    } catch (SecurityException | ClassNotFoundException e) {
            e.printStackTrace();
         }
    }

}
```

### 参考资料
* <a href="http://product.dangdang.com/9317290.html" target="_blank"><b>《Java 编程思想 Thinking in Java》</b></a>，Bruce Eckel 著，陈昊鹏 译
* 代码样例 [来源](http://ifeve.com/java-annotations/)

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang [AT] foxmail.com
* 封面图片来自<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px;" href="http://unsplash.com/@berry807?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Berry van der Velden"><span style="display:inline-block;padding:2px 3px;"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-1px;fill:white;" viewBox="0 0 32 32"><title></title><path d="M20.8 18.1c0 2.7-2.2 4.8-4.8 4.8s-4.8-2.1-4.8-4.8c0-2.7 2.2-4.8 4.8-4.8 2.7.1 4.8 2.2 4.8 4.8zm11.2-7.4v14.9c0 2.3-1.9 4.3-4.3 4.3h-23.4c-2.4 0-4.3-1.9-4.3-4.3v-15c0-2.3 1.9-4.3 4.3-4.3h3.7l.8-2.3c.4-1.1 1.7-2 2.9-2h8.6c1.2 0 2.5.9 2.9 2l.8 2.4h3.7c2.4 0 4.3 1.9 4.3 4.3zm-8.6 7.5c0-4.1-3.3-7.5-7.5-7.5-4.1 0-7.5 3.4-7.5 7.5s3.3 7.5 7.5 7.5c4.2-.1 7.5-3.4 7.5-7.5z"></path></svg></span><span style="display:inline-block;padding:2px 3px;">Berry van der Velden</span></a>
 