---
layout: post
title: "谈谈在 Java 中实现定时任务的几种方式"
subtitle: ""
date: 2018-07-17
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 活学活用
---

工作中经常接触到定时任务，实现定时任务的方式很多，常见的有 Spring @schedule 注解配合 Cron 表达式、JDK 自带的 TimerTask or Timer、使用开源作业调度框架 Quartz、线程池 ScheduleExecutorService 和其实现类 ScheduledThreadPoolExecutor。

## @schedule 注解
Spring 中的 @schedule 是我最早接触学会的，注解实现非常方便，注解中有三个参数分别是 Cron 表达式、fixedRate 和 initialDelay，具体使用看你的场景，需要注意的是使用 @schedule 注解的方法的类必须用 @Component （最好）或者 @Service @Controller 等注解进行修饰，将其交给 Spring 进行托管，且在 XML 配置文件中必须有相应的配置。随着学习的深入，后来了解到 @schedule 内部本质上是基于 Quartz 实现的。Quartz 是一个任务调度框架，@schedule 注解修饰的方法会被交给 ScheduledAnnotationBeanPostProcessor 处理，在 ScheduledAnnotationBeanPostProcessor 类中会解析 @schedule 中的三个参数，根据参数的不同，将任务交给 ScheduledTaskRegistrar 进行处理，3种不同属性的task均由quartz的taskScheduler的不同方法来完成，scheduleWithFixedDelay、
scheduleAtFixedRate 和 schedule。

## Timer & TimerTask 
Timer 是 JDK 中提供的一个定时器工具，使用的时候会在主线程之外起一个单独的线程执行指定的计划任务，可以指定执行一次或者反复执行多次。TimerTask 是一个实现了 Runnable 接口的抽象类，代表一个可以被 Timer 执行的任务。我们可以这样理解 Timer 是一种定时器工具，用来在一个后台线程计划执行指定任务，而 TimerTask 是一个抽象类，它的子类代表一个可以被 Timer 计划的任务。

TimerTask 我自己极少使用，主要是它有一些弊端，Timer 在执行定时任务时只会创建一个线程，所以如果存在多个任务，且任务时间过长，超过了两个任务的间隔时间，那么第二个任务只能等待第一个任务执行完毕后再执行，其行为可能已经不符合编码者期望的了。例如 t2 定义 3s 后开始执行，t1 定义在 1s 后执行，但是 t1 执行过程总共消耗了 4s，那么实际上 t2 会在第 6s 才会执行，因为 Timer 是单线程的缘故。

```java
package com.zhy.concurrency.timer;
import java.util.Timer;
import java.util.TimerTask;

public class TimerTest
{
	private static long start;
	public static void main(String[] args) throws Exception{
		TimerTask task1 = new TimerTask(){
			@Override
			public void run(){
				System.out.println("task1 invoked ! " + (System.currentTimeMillis() - start));
				try{
					Thread.sleep(3000);
				} catch (InterruptedException e){
					e.printStackTrace();
				}
 
			}
		};
		TimerTask task2 = new TimerTask(){
			@Override
			public void run(){
				System.out.println("task2 invoked ! " + (System.currentTimeMillis() - start));
			}
		};
		Timer timer = new Timer();
		start = System.currentTimeMillis();
		timer.schedule(task1, 1000);
		timer.schedule(task2, 3000);
	}
}

```
另外由于单线程架构，一旦 Timer 抛出运行时异常，那么 Timer 将停止所有的任务执行。Timer 还依赖于系统时间，而非时间延迟，因此一旦系统时间发生改变，可能造成执行行为的不可预测。关于 Timer 和 TimerTask 有时间的话写个源码分析仔细看看。

## Quartz
Quartz 相比写程序的人应该都比较熟悉了，名声在外。如果你想在某一个有规律的时间点干某件事，并且时间的触发的条件可以非常复杂（比如每月最后一个工作日的17:50），复杂到需要一个专门的框架来干这个事。 Quartz就是来干这样的事，你给它一个触发条件的定义，它负责到了时间点，触发相应的Job起来干活。

### Quartz 特点
但是说来惭愧，我自己也还没玩过。。。所以趁这个机会学习一下。
作为一个优秀的开源调度框架，Quartz 具有以下特点：
1. 强大的调度功能，例如支持丰富多样的调度方法，可以满足各种常规及特殊需求；
2. 灵活的应用方式，例如支持任务和调度的多种组合方式，支持调度数据的多种存储方式；
3. 分布式和集群能力，Terracotta 收购后在原来功能基础上作了进一步提升。
4. 作为 Spring 默认的调度框架，Quartz 很容易与 Spring 集成实现灵活可配置的调度功能。

### Quartz 调度核心元素：
1. Scheduler : 任务调度器，是实际执行任务调度的控制器。在spring中通过SchedulerFactoryBean封装起来。
2. Trigger : 触发器，用于定义任务调度的时间规则，有SimpleTrigger,CronTrigger,DateIntervalTrigger和NthIncludedDayTrigger，其中CronTrigger用的比较多，本文主要介绍这种方式。CronTrigger在spring中封装在CronTriggerFactoryBean中。
3. Calendar : 它是一些日历特定时间点的集合。一个trigger可以包含多个Calendar，以便排除或包含某些时间点。
4. JobDetail : 用来描述Job实现类及其它相关的静态信息，如Job名字、关联监听器等信息。在spring中有JobDetailFactoryBean和 
MethodInvokingJobDetailFactoryBean两种实现，如果任务调度只需要执行某个类的某个方法，就可以通过MethodInvokingJobDetailFactoryBean来调用。
5. Job : 是一个接口，只有一个方法void execute(JobExecutionContext context),开发者实现该接口定义运行任务，JobExecutionContext类提供了调度上下文的各种信息。Job运行时的信息保存在JobDataMap实例中。实现Job接口的任务，默认是无状态的，若要将Job设置成有状态的，在quartz中是给实现的Job添加@DisallowConcurrentExecution注解（以前是实现StatefulJob接口，现在已被Deprecated）,在与spring结合中可以在spring配置文件的job detail中配置concurrent参数。

### 使用例子
```java
import java.util.Date;
import org.quartz.DisallowConcurrentExecution;
import org.quartz.Job;
import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

public class TestQuartz implements Job {
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDetail detail = context.getJobDetail();
        String name = detail.getJobDataMap().getString("name");
        System.out.println("HHH");
    }
}
```
```java
import static org.quartz.DateBuilder.newDate;
import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;
import static org.quartz.TriggerBuilder.newTrigger;

import java.util.GregorianCalendar;

import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;
import org.quartz.impl.calendar.AnnualCalendar;

public class QuartzTest {
    public static void main(String[] args) {
        try {
            //创建scheduler
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            //定义一个Trigger
            Trigger trigger = newTrigger().withIdentity("trigger1", "group1") //定义name/group
                .startNow()//一旦加入scheduler，立即生效
                .withSchedule(simpleSchedule() //使用SimpleTrigger
                .withIntervalInSeconds(1) //每隔一秒执行一次
                .repeatForever()) //一直执行，奔腾到老不停歇
                .build();

            //定义一个JobDetail
            JobDetail job = newJob(TestQuartz.class) //定义Job类为HelloQuartz类，这是真正的执行逻辑所在
                .withIdentity("job1", "group1") //定义name/group
                .usingJobData("name", "quartz") //定义属性
                .build();

            //加入这个调度
            scheduler.scheduleJob(job, trigger);

            scheduler.start();
            Thread.sleep(50000);
            scheduler.shutdown(true);
        } catch (Exception e) {
        }
    }
}
```

## ScheduleExecutorService
ScheduleExecutorService 本身是一个接口，继承自 ExecutorService，里面有这样几个方法：
```java
//每次延时 delay 执行 Runnable 任务
public ScheduledFuture<?> schedule(Runnable command,long delay, TimeUnit unit);
//Callable 可以拿到回调的结果
public <V> ScheduledFuture<V> schedule(Callable<V> callable,long delay, TimeUnit unit);
//看名字，固定速率执行，跟Timer不一样
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
//注意，这个是固定延迟，一个任务执行完毕之后延迟 delay 再执行
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

//使用方法，一般两种，这两者内部其实都调用了 ScheduledThreadPoolExecutor
//Executors其实是个工具类，里面提供了好多静态方法，根据用户选择返回不同的线程池实例。
ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
ScheduledExecutorService executor = Executors.newScheduledThreadPool(threadNum);
```

## ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor 继承自 ThreadPoolExecutor 并实现了 ScheduleExecutorService 接口。使用时可以直接 new 一个线程池出来然后调用相应的方法。



