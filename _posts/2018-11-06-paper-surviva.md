---
layout: post
title: "Survivability of Cloud Databases - Factors and Prediction，SIGMOD 18"
subtitle: "论文阅读系列"
date: 2018-11-06
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags: 
    - 论文阅读
    - 分布式系统
---

这是 `SIGMOD 18` 新设的 `industry section` 收录的一篇来自微软 `Azure` 团队的论文，主题是通过检测数据`（telemetry raw data）`来预测数据库是 `short-lived`（<= 30 天）还是 `long-lived`（>30天），他们认为理清这个有助于他们了解用户行为和帮助他们进行更好地资源隔离、分配和调度，从而提高收入（我觉得是根本目的吧）。

![](https://pic3.zhimg.com/80/v2-7d1f3af7f2b782b5da3df810d389a6de_hd.jpg)

预测的方式是从检测数据中提取特征`（features）`，然后运用机器学习中的随机森林算法（借助 `python` 的 `scikit-learn` 来进行分析）。

最开始他们将样本通过地理区域的不同划分为三个 `region`，然后将每个 `region` 中包含的数据库划分为三类，`Basic`、`Standard` 和 `Premium`（这部分数据集 80% 为训练集，剩余 20 % 测试集）。

谈到抽取何种 `feature`，我认为应该是本篇的难点，但是实际上文章只是一带而过，告诉你我们主要有以下 `feature`：

1. `Creation Time`
2. `Server and database names`
3. `Edition and performance level`
4. `Subscription type`
5. `Subscription history`

通过分析的结果绘制 `KM` 生存曲线：

![](https://pic2.zhimg.com/80/v2-4d1aa628d41d8e8034e956b7c492c269_hd.jpg)

然后是一些置信度计算来说明算法的准确率很高（略过这部分了，本身算法也不是很复杂）

最后，他们得出的结论是：算法的准确率在 90% 以上，他们可以借助结论来规划资源分配和通过资源池对不同类别生存时长的数据库进行资源隔离；而抽取的若干特征中最值得关注的前三位是：

1. `Subscription history`
2. `Server and database names`
3. `Creation Time`

总结下来，论文主要是经验之谈，没什么让人眼前一亮的玩法，优势就在于 `Azure` 平台有丰富的历史数据可以做这种分析。

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>