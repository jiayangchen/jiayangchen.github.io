---
layout: post
title: "Java 中的静态代理与动态代理"
subtitle: "RPC 相关第五篇"
date: 2018-10-05
author: "ChenJY"
header-img: "img/java.jpg"
catalog: true
tags: 
    - 设计模式
    - Java Tech
    - RPC
---

## 什么是代理模式
人话来讲就是由代理对象来执行目标对象的方法，且还可以在代理对象中增强目标对象方法的一种设计模式。类比生活，像是房产中介。代理模式存在的意义和一个架构设计原则息息相关 —— 开闭原则（对扩展开放，对修改关闭），即一种好的设计模式，都是在不修改原有形态的基础上扩展出新的功能。

## 为什么需要代理
代理模式的概念很容易理解，但是早期的我即使读懂了代理模式的概念，对为什么要使用代理模式，还是一头雾水。为什么我不直接调用目标对象的方法，非得要借助个代理对象呢？

### 1. 调用的目标对象在远程主机上，并不在你本地
类似中介就是房东出国了，联系不上，你只能跟我沟通。对应到我们程序设计的时候就是：客户端无法直接操作实际目标对象。为什么无法直接操作？一种情况是你需要调用的对象在另外一台机器上，你需要跨越网络才能访问，如果让你直接编码实现远程调用，你需要处理网络连接、处理打包、解包等等非常复杂的步骤，所以为了简化客户端的处理，我们使用代理模式，在客户端建立一个远程目标对象的代理，客户端就象调用本地对象一样调用该代理，再由代理去跟实际对象联系，对于客户端来说背后这个通信过程是透明的。

### 2. 你不想理会目标类繁杂的功能，只希望增加一些自己的行为进去
例如常见的例子就是 Spring AOP 实现日志功能，你不必关心目标类究竟如何繁杂，你只是想要在前后调用的时候打印一下日志，那么你就可以使用代理模式，通过 AOP 提供的切面进行编码实现，你通过代理模式达到了在目标对象的方法前后增加了一些自定义行为的目的。类似的例子还有权限校验。这样做的好处有很多，一方面你需要在意目标类的代码，二来你维护了目标类功能的单一性，而不是将日志或者权限校验的功能硬编码到目标类的方法中。

## 静态代理
静态代理非常简单，就是实现类和代理类均实现同样的接口，然后在代理类中通过构造器将接口或者实现类注入进来，然后就可以在代理类的方法实现中增加一些自己的逻辑。看个 [例子](https://www.cnblogs.com/daniels/p/8242592.html) 就懂了：

### 静态代理的例子

```java
// 接口
public interface BuyHouse {
    void buyHosue();
}

// 实现类
public class BuyHouseImpl implements BuyHouse {
    @Override
    public void buyHosue() {
        System.out.println("HHHHH");
    }
}

// 代理类
public class BuyHouseProxy implements BuyHouse {
    
    private BuyHouse buyHouse;
    // 将接口引入
    public BuyHouseProxy(final BuyHouse buyHouse) {
        this.buyHouse = buyHouse;
    }

    @Override
    public void buyHosue() {
        // 增加一些自己的逻辑
        System.out.println("HHHHH");
        buyHouse.buyHosue();
        System.out.println("66666");

    }
}
```

### 静态代理的缺点
很明显，静态代理中，一个代理类只能对一个业务接口的实现类进行包装，如果有多个业务接口的话就要定义很多实现类和代理类才行。而且，如果代理类对业务方法的预处理、调用后操作都是一样的（比如：调用前输出提示、调用后自动关闭连接），则多个代理类就会有很多重复代码。这时我们可以定义这样一个代理类，它能代理所有实现类的方法调用：根据传进来的业务实现类和方法名进行具体调用。即动态代理模式。Java 中常见的有 JDK 动态代理和 CGLib 动态代理，前者只能代理接口，后者可以代理实现类。

## JDK 动态代理
JDK 的动态代理使用到 Java reflect 包下的 Proxy 类和 InvocationHandler 接口。

```java
public class DynamicProxyHandler implements InvocationHandler {

    private Object object;

    public DynamicProxyHandler(final Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("HHHHH");
        Object result = method.invoke(object, args);
        System.out.println("66666");
        return result;
    }
}

public class DynamicProxyTest {
    public static void main(String[] args) {
        BuyHouse buyHouse = new BuyHouseImpl();
        BuyHouse proxyBuyHouse = (BuyHouse) Proxy.newProxyInstance(BuyHouse.class.getClassLoader(), new Class[]{BuyHouse.class}, new DynamicProxyHandler(buyHouse));
        proxyBuyHouse.buyHosue();
    }
}
```

DynamicProxyHandler 实现了 InvocationHandler 接口，并复写其 invoke 方法，我们可以看到 invoke 方法的参数是实现类和方法参数列表。测试类中通过 newProxyInstance 这个静态工厂方法创建了代理对象，代理对象的每个执行方法都会替换执行InvocationHandler 中的 invoke 方法。这个方法总共有3个参数：

1. ClassLoader loader用来指明生成代理对象使用哪个类装载器
2. Class<?>[] interfaces用来指明生成哪个对象的代理对象，通过接口指定，这就是为什么 JDK 动态代理必须要通过接口的方式
3. InvocationHandler 用来指明产生的这个代理对象要做什么事情。

newProxyInstance 内部本质上是根据反射机制生成了一个新类。

## CGLib 动态代理
CGLib 是针对类来实现代理的，原理是对指定的实现类生成一个子类，并覆盖其中的方法实现代理。因为采用的是继承，所以不能对final 修饰的类进行代理。[例子](http://www.cnblogs.com/ygj0930/p/6542259.html) 如下：

```java

// 实现类，有没有实现接口无所谓
public class BookFacadeImpl {  
    public void addBook() {  
        System.out.println("新增图书...");  
    }  
}  

public class BookFacadeCglib implements MethodInterceptor {  
    // 业务类对象，供代理方法中进行真正的业务方法调用
    private Object target;
  
    // 相当于JDK动态代理中的绑定
    public Object getInstance(Object target) {  
        // 给业务对象赋值
        this.target = target;  
        // 创建加强器，用来创建动态代理类
        Enhancer enhancer = new Enhancer(); 
        // 为加强器指定要代理的业务类（即：为下面生成的代理类指定父类）
        enhancer.setSuperclass(this.target.getClass());  
        // 设置回调：对于代理类上所有方法的调用，都会调用CallBack，而Callback则需要实现intercept()方法进行拦
        enhancer.setCallback(this); 
       // 创建动态代理类对象并返回  
       return enhancer.create(); 
    }
    // 实现回调方法 
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) 
    throws Throwable { 
        System.out.println("预处理——————");
        //调用业务类（父类中）的方法
        proxy.invokeSuper(obj, args); 
        System.out.println("调用后操作——————");
        return null; 
    } 
}

// 测试类
public static void main(String[] args) {      
    BookFacadeImpl bookFacade = new BookFacadeImpl()；
    BookFacadeCglib cglib = new BookFacadeCglib();  
    BookFacadeImpl bookCglib = (BookFacadeImpl)cglib.getInstance(bookFacade);  
    bookCglib.addBook(); 
}  
```

## 参考资料

1. [Java动态代理之JDK实现和CGlib实现（简单易懂）](https://www.cnblogs.com/ygj0930/p/6542259.html)
2. [java的动态代理机制详解](http://www.cnblogs.com/xiaoluo501395377/p/3383130.html)
3. [java动态代理实现与原理详细分析](https://www.cnblogs.com/gonjan-blog/p/6685611.html)

## License
* 本文遵守创作共享 [CC BY-NC-SA 3.0协议](https://link.zhihu.com/?target=https%3A//creativecommons.org/licenses/by-nc-sa/3.0/cn/)



