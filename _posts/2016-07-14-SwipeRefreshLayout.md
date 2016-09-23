---
layout: post
title: Google 官方下拉刷新组件 SwipeRefreshLayout 的使用
date: 2016-07-14
categories: blog
tags: [Android]
description: 
---

　　虽然发现网上有很多包装好的下拉刷新框架，但是大多不易配置，或者是使用的时候嵌套进自己的功能就容易出现各种问题，对于经验老道的开发者来说，这些可能都不是事儿。但对于我这样刚接触Android开发两星期的小白来说，使用这些包装的框架反而并没有给我带来便利，找来找去也没找到解决办法，直到转向 Google 官方提供的下拉刷新组件 __SwipeRefreshLayout__，才发现挺容易上手的，虽然扩展性、定制性肯定比不上那些大神为你包装好的，但是对于实现简单的下拉刷新功能来说却是足够了。

　　下面开始介绍，其实很短。。。

### Introduction

　　SwipeRefreshLayout是谷歌公司推出的用于下拉刷新的控件，SwipeRefreshLayout已经被放到了sdk中，在Version 19.1之后SwipeRefreshLayout 被放到support v4中。

### Usage

　　其使用步骤非常简单：

* 第一步，先在 activity_main.xml 中添加 SwipeRefreshLayout 组件

```java

<android.support.v4.widget.SwipeRefreshLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/swipe_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="wrap_content" >

        <!-- 这里用TextView举例，更一般的是ListView，如果使用ListView 记得去掉 ScrollView 标签 -->
        <TextView
            android:id="@+id/textView1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:gravity="center"
            android:paddingTop="10dp"
            android:text="@string/swipe_to_refresh"
            android:textSize="20sp"
            android:textStyle="bold" />

    </ScrollView>

</android.support.v4.widget.SwipeRefreshLayout>

```

* 第二步在Activity中

```java
tv = (TextView)findViewById(R.id.textView1);
        swipeRefreshLayout = (SwipeRefreshLayout)findViewById(R.id.swipe_container);
        //设置刷新时动画的颜色，可以设置4个
        swipeRefreshLayout.setColorSchemeResources(

        android.R.color.holo_blue_light, 
        android.R.color.holo_red_light, 
        android.R.color.holo_orange_light, 
        android.R.color.holo_green_light);

        swipeRefreshLayout.setOnRefreshListener(new OnRefreshListener() {
            
            @Override
            public void onRefresh() {
                tv.setText("正在刷新");
                // TODO Auto-generated method stub
                new Handler().postDelayed(new Runnable() {
                    
                    @Override
                    public void run() {
                        // TODO Auto-generated method stub
                        tv.setText("刷新完成");
                        swipeRefreshLayout.setRefreshing(false);
                    }
                }, 6000);
            }
        });

```
PS: setColorScheme()已经弃用，使用setColorSchemeResources()来设置颜色。