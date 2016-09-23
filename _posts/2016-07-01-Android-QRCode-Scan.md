---
layout: post
title: Android 应用添加二维码扫描功能
date: 2016-07-01
categories: blog
tags: [Android]
description: 
---
	
　　因为暑期大作业的App里需要用到二维码扫描的功能，因此就花了半天研究了一下这个ZXing的开源库，使用还是挺简单的，好像很多App都是基于ZXing来完成扫描二维码的功能，于是今天抽空将使用方法写一下。

## 二维码

　　是用某种特定的几何图形按一定规律在平面（二维方向上）分布的黑白相间的图形记录数据符号信息的。
在代码编制上巧妙的利用构成计算机内部逻辑基础的0和1比特流的概念，使用若干个与二进制相对应的几何形体来表示文字数值信息，通过图像输入设备或光电扫描设备自动识读以实现信息自动处理。
二维码能够在横向和纵向两个方位同时表达信息，因此能在很小的面积内表达大量的信息。
二维码相对于条形码的优势就是省空间。

## ZXing简介

　　什么是ZXing呢？ZXing是一个开放源码的，用Java实现的多种格式的1D/2D条码图像处理库，它包含了联系到其他语言的端口。Zxing可以实现使用手机的内置的摄像头完成条形码的扫描及解码。

* 这是ZXing的[开源库地址](https://github.com/zxing/zxing)

　　但是，如果仅对于开发Android应用来说，我们并不需要如此这个库全部的内容，事实上，即使你使用ZXing的全部内容，你也会发现它的体积非常庞大而且很难配置。好在已经有人帮我们抽出了Android应用扫描二维码需要的文件，整合成了一个相对灵巧的模块，我们只需在自己的App中调用这个模块的Activity即可实现扫描二维码的功能。

* 这个ZXing for Android 的库文件[下载地址](https://github.com/jiayangchen/BarCodeTest)

## 其中使用时，你需要注意以下几点

* 因为你开发的App需要依赖 BarCodeTest 这个项目的二维码扫描与生成的功能，即 BarCodeTest 这个项目是作为一个依赖模块出现在你的工程中的，与你自己开发的App是从与主的关系。

* 因此 Eclipse 里记得在 Import Project 之后，右键项目选择 Properties -> Android 勾选 Project Build Target 与 is Library 选项，使其成为一个 Library

* Android 里 File -> Import Module，然后参见这篇[博文](http://blog.163.com/benben_long/blog/static/19945824320151117103412653/)，写的非常棒！

* 最后，当IDE报出常量错误的时候，不要怕，跳到出错位置你会发现是由于Switch语句的Case部分报错，这仅当原本的项目修改为library之后才会出现，我们所要做的是将Switch语句用if 、else if语句替换即可。

## 结尾

　　至此，你已经可以在开发的App中使用BarCodeTest里的功能了，指定响应跳转至BarCodeTestActivity或CaptureActivity即可。

* 文章最后，放上极客学院对于ZXing for Android的一段[免费教程](http://www.jikexueyuan.com/course/134_1.html?ss=1)，对本文内容有任何疑问的可以在视频中进一步学习。