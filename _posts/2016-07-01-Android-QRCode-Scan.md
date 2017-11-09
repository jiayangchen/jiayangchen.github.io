---
layout: post
title: "暑期大作业Android相关整理"
subtitle: "一、技术篇"
date: 2016-07-01
author: "ChenJY"
header-img: "img/android.jpg"
catalog: true
tags: 
    - Android
    - 暑期大作业
    - 技术整理
---
	
    
### 在android应用中添加扫描二维码的功能
    
　　因为暑期大作业的App里需要用到二维码扫描的功能，因此就花了半天研究了一下这个ZXing的开源库，使用还是挺简单的，好像很多App都是基于ZXing来完成扫描二维码的功能，于是今天抽空将使用方法写一下。

##### 二维码

　　是用某种特定的几何图形按一定规律在平面（二维方向上）分布的黑白相间的图形记录数据符号信息的。
在代码编制上巧妙的利用构成计算机内部逻辑基础的0和1比特流的概念，使用若干个与二进制相对应的几何形体来表示文字数值信息，通过图像输入设备或光电扫描设备自动识读以实现信息自动处理。
二维码能够在横向和纵向两个方位同时表达信息，因此能在很小的面积内表达大量的信息。
二维码相对于条形码的优势就是省空间。

##### ZXing简介

　　什么是ZXing呢？ZXing是一个开放源码的，用Java实现的多种格式的1D/2D条码图像处理库，它包含了联系到其他语言的端口。Zxing可以实现使用手机的内置的摄像头完成条形码的扫描及解码。

* 这是ZXing的[开源库地址](https://github.com/zxing/zxing)

　　但是，如果仅对于开发Android应用来说，我们并不需要如此这个库全部的内容，事实上，即使你使用ZXing的全部内容，你也会发现它的体积非常庞大而且很难配置。好在已经有人帮我们抽出了Android应用扫描二维码需要的文件，整合成了一个相对灵巧的模块，我们只需在自己的App中调用这个模块的Activity即可实现扫描二维码的功能。

* 这个ZXing for Android 的库文件[下载地址](https://github.com/jiayangchen/BarCodeTest)

##### 其中使用时，你需要注意以下几点

* 因为你开发的App需要依赖 BarCodeTest 这个项目的二维码扫描与生成的功能，即 BarCodeTest 这个项目是作为一个依赖模块出现在你的工程中的，与你自己开发的App是从与主的关系。

* 因此 Eclipse 里记得在 Import Project 之后，右键项目选择 Properties -> Android 勾选 Project Build Target 与 is Library 选项，使其成为一个 Library

* Android 里 File -> Import Module，然后参见这篇[博文](http://blog.163.com/benben_long/blog/static/19945824320151117103412653/)，写的非常棒！

* 最后，当IDE报出常量错误的时候，不要怕，跳到出错位置你会发现是由于Switch语句的Case部分报错，这仅当原本的项目修改为library之后才会出现，我们所要做的是将Switch语句用if 、else if语句替换即可。

##### 结尾

　　至此，你已经可以在开发的App中使用BarCodeTest里的功能了，指定响应跳转至BarCodeTestActivity或CaptureActivity即可。

* 文章最后，放上极客学院对于ZXing for Android的一段[免费教程](http://www.jikexueyuan.com/course/134_1.html?ss=1)，对本文内容有任何疑问的可以在视频中进一步学习。

### Google 官方下拉刷新组件 SwipeRefreshLayout 的使用

虽然发现网上有很多包装好的下拉刷新框架，但是大多不易配置，或者是使用的时候嵌套进自己的功能就容易出现各种问题，对于经验老道的开发者来说，这些可能都不是事儿。但对于我这样刚接触Android开发两星期的小白来说，使用这些包装的框架反而并没有给我带来便利，找来找去也没找到解决办法，直到转向 Google 官方提供的下拉刷新组件 __SwipeRefreshLayout__，才发现挺容易上手的，虽然扩展性、定制性肯定比不上那些大神为你包装好的，但是对于实现简单的下拉刷新功能来说却是足够了。

　　下面开始介绍，其实很短。。。

##### Introduction

　　SwipeRefreshLayout是谷歌公司推出的用于下拉刷新的控件，SwipeRefreshLayout已经被放到了sdk中，在Version 19.1之后SwipeRefreshLayout 被放到support v4中。

##### Usage

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

### Android在一个app程序中，打开另一个app的方法

暑期大作业里，碰到了需要调用外部第三方App的情况，那么如何在你自己开发的这个app中启动手机里装好的另一个app呢？我一开始查找的方法中，我一直以为包名应该是build.gradle文件中的application id，那实际上却不一定。

![image](http://o9oomuync.bkt.clouddn.com/open-another-appQQ%E6%88%AA%E5%9B%BE20160721143427.jpg)

##### 方法如下：

* 找到你需要打开的第三方app的安装包（apk包），导入电脑，为了方便建议存放在磁盘根目录，例如F盘根目录

![image](http://o9oomuync.bkt.clouddn.com/open-another-appQQ%E6%88%AA%E5%9B%BE20160721144001.jpg)

* 进入你下载的SDK文件夹中，依次打开SDK -> build-tools -> 23.0.3(依个人版本而不同) -> aapt.exe，看到aapt.exe之后，我们按住电脑shift键同时点击鼠标右键，在菜单中选择在此处打开命令行窗口

![image](http://o9oomuync.bkt.clouddn.com/open-another-appQQ%E6%88%AA%E5%9B%BE20160721144344.jpg)

* 输入命令：aapt dump badging F:\me.chenjiayang.myapplication.apk，在命令行窗口中会显示此apk包的相关信息，上翻找到package name="xxx"，这里的xxx就是我们完成功能所需要的包名

![image](http://o9oomuync.bkt.clouddn.com/open-another-appQQ%E6%88%AA%E5%9B%BE20160721144838.jpg)

* 打开android studio，在需要调用第三方app的地方输入以下代码：

```java

	PackageManager packageManager = getPackageManager();   
	Intent intent=new Intent();   
	intent = packageManager.getLaunchIntentForPackage("me.chenjiayang.myapplication");  
	startActivity(intent);  

```

### Android：利用SharedPreferences实现自动登录
* http://www.cnblogs.com/tinyphp/p/3998444.html

### 最强 Android Studio 使用小技巧和快捷键
* http://www.open-open.com/lib/view/open1458715872710.html

### android:怎么实现一个控件与另一个指定控件左对齐
* https://segmentfault.com/q/1010000003905460?_ea=425861

### 最详细的 Android Toolbar 开发实践总结
* http://www.codeceo.com/article/android-toolbar-develop.html

### 最详细的 NavigationDrawer 开发实践总结
* https://github.com/D-clock/AndroidSystemUiTraining/blob/master/note/02_AndroidSystemUI%EF%BC%9ANavigationDrawe%E7%89%B9%E6%80%A7%E7%9A%84%E4%BD%BF%E7%94%A8.md

### Android .9.png图片的制作与使用
* http://blog.csdn.net/leeccncdl/article/details/7303635

### Android开发中 ic_launcher 尺寸
* http://blog.csdn.net/never_cxb/article/details/47184241

### Android studio统计代码行数插件-->statistic
* http://blog.csdn.net/qq_23547831/article/details/50774941

### toolbar显示返回箭头
* http://blog.csdn.net/u014631944/article/details/51347117

### Android学习——百度地图开发定位与显示Demo
* http://blog.csdn.net/zhshulin/article/details/37495575#

### android应用中去掉标题栏的方法
* http://blog.csdn.net/liuzhidong123/article/details/7818531

### Android开发之高德地图实现定位
* http://www.jianshu.com/p/c3dc0cea0a2d

### Material Design Icons
* https://github.com/google/material-design-icons/releases/tag/1.0.0

### 各种Android Dialog创建及其监听事件实现
* http://blog.sina.com.cn/s/blog_4a0238270101jr0q.html

### Android学习笔记30：列表ListView控件的使用
* http://www.cnblogs.com/menlsh/archive/2013/03/15/2962350.html

### android开发教程之listview使用方法
* http://www.jb51.net/article/46652.htm

### Windows下AndroidStudio 中使用Git(AndroidStudio项目于GitHub关联)
* http://www.mamicode.com/info-detail-512945.html

### LayoutInflater的使用 和getSystemService系统服务
* http://blog.csdn.net/psuaije/article/details/7423353

### 一个千万量级的APP使用的一些第三方库
* http://www.jianshu.com/p/dc8c05cf693d

### Android项目重构之路:架构篇
* http://keeganlee.me/post/android/20150605

### 运用RUP 4+1视图方法进行软件架构设计
* http://www.uml.org.cn/zjjs/201412262.asp

### 许可协议
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
* 商业用途转载请联系 Chen.Jiayang[AT]foxmail.com
* 封面图片来自 <a href="https://unsplash.com/" target="_blank"><b> unsplash </b></a>