---
layout: post
title: "ReAct Agent：推理和工具调用为什么要交错"
subtitle: ""
date: 2026-11-18
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags:
 - AI
 - Agent
---

ReAct 的 Re 和 Act 来自 Reason 和 Act，和前端的 React 无关。模型写一段 Thought，发出一个 Action，环境把 Observation 写回上下文，再决定下一步。推理和行动交错在同一条轨迹里。

名字来自 Yao 等人 2023 年的 ICLR 论文 [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)。推理用来形成、跟踪和修改计划；行动去碰外部知识或环境，把新信息带回来。原文在 HotpotQA、FEVER 上用 Wikipedia 接口查资料，在 ALFWorld、WebShop 上按观察做决策。

## 一次只前进一跳

一轮只交出 Thought 和 Action。Thought 不改变环境，也没有新的 Observation，只把判断留在上下文里。Action 才会被执行。Observation 由执行一侧写入。停机要显式，原文用 `Finish[答案]`。模型停止调用，只说明它不想再查。

本地 `/health` 为 200、线上 404，且不许改代码：

```text
Thought: 先看请求实际打到哪。
Action: 读网关中的路径前缀
Observation: 前缀是 /api/v2，探针仍请求 /health。
Thought: 确认这是不是新改的。
Action: 看该配置最近一次提交
Observation: 昨天前缀改为 /api/v2，探针没改。
Action: Finish。说明根因，没改文件。
```

第二次 Thought 用的是第一次 Observation。没有它，下一步只能继续猜。

## 和另两种做法的差别

Chain-of-thought 把推理一次写完。材料已在上下文里时，这样更合适。材料在外面时，没读过配置也可以写成「nginx 重写了 `/health`」，错误会往下传。观察让下一步看见工具结果。

只行动、不写推理时，空结果分不清「没有路由」和「搜错目录」。论文图示里，只行动的轨迹拿到检索结果，仍可能得不出该结束的那一步。

先写完计划再调用工具，也不等于 ReAct。计划若在观察返回前写完，后面只是执行它。这个循环要求每一跳按已写回的上下文重选下一步，包括承认上一步过时。

流程的步骤和失败分支在运行前写定。ReAct 的下一步取决于上一条 Observation，同一次提问可以走出不同轨迹。轨迹记录的是这次做了什么。发布、迁移这类顺序固定的事，控制流放在循环外；内部还要看返回值时，再用这个循环。

## 循环怎么坏

原地打转：判断没变，却重复同一个 Action。论文把跳不出旧 thought 和 action 归为推理错误。步数上限和重复检测要在模型外。HotpotQA、FEVER 上他们以 7 步和 5 步为界，给不出答案就后退，并写道更多步数没有提高表现。这两个数不能当通用配置。

观察缺失：结果还没返回就再次调用，或失败被丢掉，或收下模型自己写的 `Observation:`。最后一种格式像完整轨迹，依据是猜的。

工具错误：超时、非零退出、空结果、错误页都不是成功。空是事实，失败不是事实。返回要区分有内容、为空、失败，并注明副作用。Thought 写着只读、Action 却在写文件时，权限看 Action。没有任何 Observation 就 Finish，应拒绝，并把拒绝写回去。

## 什么时候合适

下一步取决于还没看到的状态，且以读和可撤销的写为主，这时合适。材料已齐全时，一次 chain-of-thought 更干净。不可逆动作由循环外的权限决定能否执行。任务长到会挤掉初始约束时，进度放在上下文外，循环只做当前这一小段。长调用会堵住同步回路，应拆到后台。

Observation 必须进入下一次生成。Thought 若只留在单独的推理通道、没有写回，下一步就不记得这一跳的原因。

## 参考

- Yao, Shunyu 等. [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629). ICLR 2023. 排查例子和实现判断不是论文实验。

## License
* 本文遵守创作共享 CC BY-NC-SA 3.0协议
