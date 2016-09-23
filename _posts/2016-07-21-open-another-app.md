---
layout: post
title: Android在一个app程序中，打开另一个app的方法
date: 2016-07-21
categories: blog
tags: [Android]
description: 
---

暑期大作业里，碰到了需要调用外部第三方App的情况，那么如何在你自己开发的这个app中启动手机里装好的另一个app呢？我一开始查找的方法中，我一直以为包名应该是build.gradle文件中的application id，那实际上却不一定。

![image](http://o9oomuync.bkt.clouddn.com/open-another-appQQ%E6%88%AA%E5%9B%BE20160721143427.jpg)

### 方法如下：

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
