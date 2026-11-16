---
layout: post
title: "为什么近期 Agent 开发都在转向异步执行"
subtitle: ""
date: 2026-11-16
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags:
    - AI
    - Agent
---

最近读了一批英文工程博客、官方文档，以及 X 上的讨论，一个判断几乎同时出现：Agent 开发正在从同步的对话轮次循环，转向异步、可后台运行的执行方式。这不是 API 风格从同步换成 `async/await` 的时髦写法，而是产品和系统层面的转向——把面向用户的 Agent 空出来，让它随时能对话、纠偏和取消，把耗时长的工作丢到后台，并且跑得够久、挂了还能续上。

之所以说「最近」而且「都不约而同」，核心不在框架之争，而在于一件事终于成立了：模型可以较少人工盯着，就把一份规格做到接近可合并的代码。同步的 ReAct 循环撑不住这种工作时长，于是编排、产品形态和运行时一起改了形状。

## 「异步」其实不是一件事

讨论里说的 async，至少叠了几层相近、但并不相同的含义。

第一层是后台、长周期 Agent。工作在一轮对话结束后继续跑，用户不必守着终端或笔记本。Cursor 的 Cloud Agents、OpenAI 的 Dots、Claude Code 的后台会话，都属于这一类。

第二层是真正的非阻塞工具调用。模型发出工具调用后立刻拿到回执或任务 ID，可以继续说话或做别的事，结果稍后再作为新消息回来。OpenAI 的 Async tool calling 文档，以及 Mike Chambers 在 2026 年 2 月的文章，都特别强调：这和「一次发出多个工具、仍要等最慢的那个返回」不是一回事。

第三层是并行工具调用，或并发的子 Agent。互相独立的只读调用可以扇出，延迟取最大值而不是求和。Anthropic 的并行工具使用文档讲的就是这一层。它很有用，但循环依然会堵住，直到最慢的调用结束。

第四层是耐用、事件驱动的工作流。任务可能跑几小时，也可能等人工审批几天、等外部系统几周。进程不能一直挂着，崩溃后要能从上次成功的步骤恢复。Temporal、Inngest，以及 Kiran Prakash 发表在 martinfowler.com 上的长文，都把这一层当成分布式系统问题，而不是聊天接口问题。

第五层是跨会话的 harness。Anthropic 在 2025 年 11 月的研究指出，数小时甚至跨天的工作装不进一次上下文，压缩也不够用，必须靠进度文件、git 提交和功能清单，让下一轮会话接着干。即便忽略超时和交互体验，「永远开着同一个同步循环」也不是完整答案。

LangChain 在 2026 年 11 月 5 日的文章 *Agents Are Going Async* 里，把生态正在收敛的形态说得很清楚：面向用户的编排器保持可响应，长工作交给后台工人。

## 为什么是现在

最好的「为什么是最近」的回答，不是哪一家突然发明了异步 API，而是模型跨过了可靠度门槛。

Latent Space 在 2026 年 5 月 28 日那期 *The Age of Async Agents* 里，把拐点放在 2025 年 12 月前后。OpenInspect 的 Cole Murray 提到，Opus 4.5 和 GPT 5.2 让团队不再需要一路手把手带着模型走；只要规格足够清楚，从规格做到可合并的 Pull Request，摩擦已经小了很多。Cognition 的 CPO Walden Yan 也说，模型开始达到可以放手的自主程度，争论从「要不要一直待在 IDE 里盯着」变成了「能不能直接丢到云端」。

同一篇文章也指出，异步或后台 Agent 在 2024 年还是一次大胆的押注，因为当时模型还不够好，人们也不放心让它独自跑完。Cognition 的 Devin、Cursor 的后台或云端 Agent 出现得更早，但整个栈真正跟上来，是在模型能够较为可靠地独自收尾之后。

Cursor 自己在 2026 年 7 月的一则帖子里给出了一组产品数字：2025 年 12 月，他们合入的 Pull Request 里只有十分之一来自 Cloud Agents；到 2026 年 7 月，这个比例到了 56%。他们的解释是，给 Agent 独立的云端环境，让它把更长的任务从头跑到尾。这组数字来自官方声明，我没有再核对内部统计口径，但方向和上面的能力拐点是对得上的。

## 任务已经超出一轮对话

更早的 Agent 大多是同步、聊天式的，因为它们必须足够快。Inngest 在 2026 年 5 月的文章里写得很直白：现在的 Agent 会承担持续数分钟到数小时的长周期任务，循环可以跨越上百步；成功的应用几乎都需要异步后台工作，因为任务已经太长，同步交互体验撑不住。

OpenAI 的 Background mode 文档说得更工程化：像 Codex 和 Deep Research 这样的 Agent 说明，推理模型可能要跑好几分钟。后台模式让长任务不必担心超时或连接中断，请求以 `background: true` 启动，客户端再轮询或恢复数据流。

LangChain 在 2026 年 4 月已经写过更具体的产品后果：内联子 Agent 适合小活，可是「如果一个子 Agent 要跑一个小时，你就得等一个小时才能再跟主 Agent 说话」。Fowler 站点上的文章把时间尺度又拉长了一档：长周期 Agent 可能等两天的人工审批，再等三周的外部系统。一个必须一直活着的进程，不是可行的设计。

## 人还在，Agent 就得叫得应

如果只是把任务丢到后台，却让面向用户的那一层消失，体验同样会坏掉。LangChain 11 月那篇文章开头用了一个比喻：一位每次接到大项目就失踪的幕僚长，是会被开除的。Agent 必须同时做到两件事：立刻响应新的指令，并且带着先前的上下文；同时能并行推进多项大任务。标准 ReAct 循环做不到这一点，因为一次长工具调用就会堵住整个回路，其他任务只能排队。

他们观察到的收敛形态，是「一个跟用户待在一起的 Agent，加上做长活的后台工人」。Claude Code 的后台子 Agent、OpenAI 的 Dots，走的是同一条路。Cursor 在 2025 年 10 月介绍 Cloud Agents 时也强调，可以同时跑很多个 Agent，不必让笔记本保持在线：修一个 bug 而不必离开手头的工作，午饭或通勤前把待办丢出去，或者把本地计划交给云端 Agent 去实现。

Mike Chambers 补充了一个更尖锐的场景：即便并行调用了多个工具，只要最慢的那个要 45 秒，前面 44 秒里模型仍然是静音的。真正的异步工具调用，对聊天、尤其是语音交互很关键——「冷场对语音体验是致命的」。

## 吞吐要靠工人，而不是把一条循环拉得更长

LangChain 的表述是：Agent 能承担的工作量，应该随后台工人的数量增长，而不是随单条循环的长度增长。被堵住的监督者既不能和用户说话，也不能再启动新工作，更无法在中途纠偏。所以异步子 Agent 不该是「发出去就不管」，而应是「发出去还能转向」：可以启动、更新、取消、列出，并在完成时得到通知。

并行工具调用仍然是有价值的积木，但只解决「同一轮里独立调用不要串行相加」。它不能代替把长任务从用户回路里拆出去。

## 跑得越久，崩溃恢复就越不可少

任务变长之后，失败的代价会陡然上升。Inngest 的核心论点是：四五十分钟的 Agent 运行，不能活在会超时的无服务器函数里，也不能只活在单个进程的内存里。后台 Agent 还需要多步骤的可观测性、由事件唤醒的暂停与恢复（人工审批、webhook、另一个 Agent），以及状态查询、取消和调度这些生命周期控制。

Temporal 在 2026 年 7 月介绍 LangGraph 插件时，把检查点和耐用执行区分得很清楚：LangGraph 会保存检查点，但检查点不等于耐用执行。一次运行如果只活在一个进程里，进程没了，运行也就没了。人工审查可能只要三十秒，也可能要三天；把进程挂在那里浪费算力，拆掉后又得自己做恢复。耐用等待（信号、定时器）在等待期间几乎不消耗计算，而且暂停可以只作用在需要审查的那条分支上，图的其余部分继续往前走。

Kiran Prakash 的文章把长周期 Agent 直接当成分布式系统：进程可能在等待中死亡，远程调用可能失败，至少一次投递的重试可能重复提交，并发也会带来竞态。耐用执行记录每一步外部操作，再通过回放重建状态。等待不必让进程一直活着。他们的测量是：一个正在等待的工作流不占用 CPU。

人工在环必须是异步的，否则整个 Agent 都会冻住。OpenAI 的异步工具调用也允许模型一边向用户提问，一边继续做相互独立的工作，而不必把一次响应一直开着等回复。Dots 则在产品层规定了何时自行推进、何时请求批准：只读的主动研究可以自己做，有后果的动作要经过审查。

这里有一个常被忽略的边界：耐用并不等于正确。日志记录的是 Agent「看到了什么」，不是世界上「实际发生了什么」。回包丢失后再重试，可能把订单下两次，所以需要幂等；审批等了三天，价格可能已经变了，所以提交前要重新核对事实，最好给批准加上有效期或范围。

## 上下文也装不下整份工作

即使请求不会超时，上下文窗口也未必撑得住。Anthropic 的 Justin Young 在 *Effective harnesses for long-running agents* 里写过一组失败模式：Agent 试图一次性做完整件应用，实现到一半上下文就用完了；后一轮会话过早宣布完工；功能被标成完成，却没有端到端测试。压缩解决不了「每一轮新会话都不记得从前」。他们的修法是一套 harness：初始化 Agent、功能清单、git 提交和进度文件，让下一轮可以增量接着做。

这也解释了为什么「把一个同步循环永远开着」即使用户体验和超时都不管，仍然不够。

## 开发者实际怎么用，以及何时不该异步

X 上的讨论和文章是对齐的，但更偏实际用法。Geoffrey Litt 把一种工作流称为 “coding like a surgeon”：用慢而强的异步 Agent 隔夜做调研、写文档、预演改动，这些结果只作为松散参考，并不直接拿去合入；真正动手时再用快速的同步工具保持心流，尽量不要在现场会话里踢出慢任务，因为等待会把人拉开。他的理由很朴素：复杂工作必须自己理解，而理解最快的办法往往是参与把东西做出来。异步预演就算大致做对了，从头按它走一遍也像在跟教程；如果没做对，也不必去拆一份自己看不懂的巨大 Pull Request。

swyx 则认为，把问题收成 sync 还是 async 这个二分并不准确。已有套路、相对可重复的活用廉价的后台 Agent；难的问题需要 Fast Agents，把人留在心流窗口里。模型实验室在追多日自主，Agent 实验室则在追人和机器之间的紧密配合。这个提醒很重要：异步不该成为所有任务的默认值。

何时应该保持同步或顺序执行，几份文档其实说得相当一致。Anthropic 指出，有副作用、共享状态或顺序要求的工具，可能必须串行。LangChain 的 Deep Agents 文档也说，当 Agent 必须先拿到结果再继续时，应使用同步子 Agent。Digital Applied 的一篇咨询文章还给出了一组更偏产品的判据：可逆性、在预期改动规模下的审查成本、是否对延迟敏感、以及「完成」能否事先定义清楚。任何一项过不了，就应更倾向同步。后台吞吐也不是免费的：N 个后台会话，配额往往也按 N 倍计。冷启动去审一份巨大的差异，人能发现的缺陷通常更少。

模型自己处理「尚未完成的工作」时也会出错：可能在等待时编造结果，可能在很多轮之后才收到乱序返回，待办在上下文中间堆积时还会「迷失在中间」。LangChain 文档里也记录过真实的监督者缺陷：从对话历史里读到过期状态、截断任务 ID、以及本该把控制权交还用户时却立刻去轮询。

沙箱快照也不是编排。Inngest 提醒过：虚拟机快照不等于工作流快照，它不会告诉你哪些步骤已经成功、怎样在不重做已完成工作的前提下恢复。短的、请求范围内的 Agent，也不一定需要 Temporal 这一级的基础设施。

## 小结

如果把这些材料收成一句判断，我会写成：这是一次产品和系统上的转向——让面向用户的 Agent 保持空闲和可响应，把长工作放到后台，并用耐用执行扛住中断。它在 2025 年底前后变得实用，是因为模型终于可以更少被手把手带着，把正经任务做完。

支撑它的理由也是一套组合，而不是单一卖点：任务超出了一轮对话和一次请求；用户仍然必须能说话、转向和取消；工作应能在多个工人之间铺开；长运行必须在崩溃和「不占计算的等待」中活下来，包括人工审批；上下文窗口装不下整份工作，所以还需要跨会话的 harness。

同时也该把边界留在手里。有依赖、不可逆、或规格说不清的工作，同步或顺序往往更合适；不要把并行工具调用误认成非阻塞的 Agent；耐用执行不会自动让决定本身变得正确。

## 参考与延伸阅读

- [Agents Are Going Async](https://www.langchain.com/blog/agents-are-going-async)，Sydney Runkle、Thushanth Bengre，LangChain，2026 年 11 月 5 日
- [Running Subagents in the Background](https://www.langchain.com/blog/running-subagents-in-the-background)，Hunter Lovell、Colin Francis，LangChain，2026 年 4 月 16 日
- [Long-horizon agents and durable execution](https://www.martinfowler.com/articles/durable-execution-agents.html)，Kiran Prakash，martinfowler.com，2026 年 11 月 10 日
- [Background agents are here. Your orchestration isn't ready.](https://www.inngest.com/blog/background-agents-are-here-your-orchestration-isnt-ready)，Inngest，2026 年 5 月 8 日
- [Effective harnesses for long-running agents](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents)，Justin Young，Anthropic，2025 年 11 月 26 日
- [Async tool calling](https://developers.openai.com/api/docs/guides/async-tool-calling)、[Background mode](https://developers.openai.com/api/docs/guides/background)，OpenAI 文档
- [Introducing dots](https://openai.com/index/introducing-dots/)，OpenAI，2026 年 9 月 29 日
- [Cloud Agents](https://cursor.com/blog/cloud-agents)，Cursor，2025 年 10 月 30 日
- [The Age of Async Agents](https://latent.space/p/cognition/)，Latent Space，2026 年 5 月 28 日
- [LangGraph in production: Temporal's LangGraph Plugin adds Durable Execution](https://temporal.io/blog/temporal-langgraph-plugin-durable-execution)，Temporal，2026 年 7 月 16 日
- [Async Agentic Tools: Breaking Free from the Request-Response Loop](https://blog.mikegchambers.com/posts/async-agentic-tools/)，Mike Chambers，2026 年 2 月 17 日
- [Geoffrey Litt：Coding like a surgeon](https://x.com/geoffreylitt/status/1990959999045005787)
- [swyx：sync/async 这个二分并不对](https://x.com/swyx/status/1983737160923804095)
- [Cursor：Cloud Agents 合入 PR 从 10% 到 56%](https://x.com/cursor_ai/status/2082841397632086241)

## License
* 本文遵守创作共享 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/cn/" target="_blank"><b>CC BY-NC-SA 3.0协议</b></a>
