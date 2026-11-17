---
layout: post
title: "Agent Harness 里的 Plan Mode 是怎么实现的"
subtitle: ""
date: 2026-11-17
author: "ChenJY"
header-img: "img/websitear.jpg"
catalog: true
tags:
 - AI
 - Agent
---

最近对照着读了一批中英文的源码解读和产品博客，想弄清一件事：各家 coding agent 所说的 Plan Mode，到底是一段「请先规划再动手」的提示词，还是一套真正的运行时？

结论比我预想的更一致。在比较成熟的 harness 里，计划模式通常不是模型的自觉，而是几样东西叠在一起：一层权限信封，周期性注入的提示词或系统提醒，一份落在磁盘上的 Markdown 计划文件，以及一个必须经过批准才能退出的闸门。提示词负责让模型「不想写」；真正保证「写不进」的，是权限系统。

[Armin Ronacher 在 2025 年 12 月 17 日的拆解](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/)已经点破了关键的一半：Claude Code 里的计划就是一份 Markdown，Edit/Write 往往还留在工具目录里，反复注入的只读提醒很重要，但审批界面靠再写一段 prompt 是复制不出来的。他当时的判断偏「几乎全是自定义提示词、提醒和 UX」。后来的源码解读把另一半补上了——工具确实还在，但权限模式已经被切走了。

## 先对齐，再动手

各家对「为什么需要计划模式」的说法几乎一样。Agent 最贵的浪费，往往不是写错代码，而是写的不是你想要的。

[Bazingga 11 月 13 日那篇源码拆解](https://jishuzhan.net/article/2121098887466057730)把检查点的粒度说得很清楚：逐次批准每一次写文件，问的是「这一刀能不能下」；先对齐方案，问的是整场手术对不对。打断从 N 次降到 1 次，但单次拦截的赌注变大。

[乔人尹在《Claude Code 解剖》](https://underestimated.cn/archives/claude-code-anatomy.html)里又往原理上收了一层：模型没有草稿纸，先把方案写出来，等于给了它一张纸；一份被批准的方案，相当于把开放式的 Agent 任务收成接近 workflow 的东西。方案阶段改一句话，比改完二十个文件再返工便宜得多。

## 共同的三层结构

把 Claude Code 相关的中英文解读放到一起看，计划模式大致是三层，而不是一句系统提示。

第一层是权限信封。进入计划模式时，harness 把 `toolPermissionContext.mode` 切到 `plan`，并记住进入前的 `prePlanMode`，退出时再恢复。

[s10 的源码解读](https://mo-jiao.github.io/claude-code-source-code/guide/s10-plan-mode.html)特别强调：这是权限模式切换，不是把 Edit、Write、Bash 从工具列表里拿掉。切换工具集会改 system prompt，进而打断 Prompt Cache 的前缀匹配；在长对话里，一次 cache miss 可能意味着数万 token 重新计算。工具还在目录里，但写操作要过权限系统。

第二层是提示词和系统提醒。进入之后，会话里会被注入只读禁令和工作流说明。Armin 和 [HP Frei Labs 在 2026 年 5 月 21 日的文章](https://hpfreilabs.com/blog/claude-code-planning-mode-vs-planning)都引过那段后来反复出现的话：Plan mode is active，除指定计划文件外不得修改系统，并且这条指令覆盖此前收到的其他指示。

[how-claude-code-works 的第 9 章](https://github.com/Windy3f3f3f3f/how-claude-code-works/blob/main/docs/10-plan-mode.md)把注入写成附件系统上的节流：不是每轮都贴完整指令。配置是每 5 轮注入一次，每 5 次注入里才有一次完整版，完整指令大约每 25 轮出现一次；压缩边界会无条件重发完整版，以免摘要把旧提醒吃掉。完整指令约 4,700 字符，精简提醒约 300 字符。软层管的是「模型别忘了自己在规划」，硬层并不依赖软层生效。

第三层是落盘的计划文件，加上退出闸门。计划不是聊天泡泡里的一段话，而是写到磁盘上的 Markdown。Claude Code 默认放在 `~/.claude/plans/` 下，文件名是随机的 word slug，也可以用 `plansDirectory` 改到仓库里。

`ExitPlanMode` 不把计划正文当作参数，而是去读这份文件；文件不存在，工具会直接拒绝。[Tim Schipper 在 2026 年 8 月 2 日对 Claude Code 2.1.220 的测试](https://tim-schipper.nl/en/blog/claude-code-plan-mode)也写过这句返回：`No plan found. Please write your plan to the plan file first.` 退出在权限管道里返回 `ask`，审批界面复用已有的权限确认，并不另造一套系统。批准后恢复的是进入前的模式，不是写死的 default。

Bazingga 还钉过一个容易说错的细节：写权限判序里并没有计划模式专属的 deny 分支。内部路径白名单（本会话的计划文件、草稿区）放行，其余写入兜底是 `ask`。所以更准确的语义不是「写工具被卸掉了」，而是「除白名单外，没有人点头，一次文件都写不进」。计划文件本身就要用 Write/Edit 写入，封死工具会误伤这条通路。

「只读」也不是把环境彻底冻住。Tim 指出，探索仍然需要 `git log`、`grep` 以及测试命令的帮助输出；从 2.1.218 起，`useAutoModeDuringPlan` 默认打开，这些命令会先交给 auto 模式的分类器，而不是每条都弹窗。计划模式拦的是未经批准的写入，不是把 shell 彻底关掉。

## 几个值得钉死的机制

进出是不对称的。模型可以自己调用 `EnterPlanMode`，效果上接近用户按 Shift+Tab；但退出必须经过审批。HP Frei 把这套信任模型说得很干脆：模型可以自己决定先规划，但不能自己决定开始实施。

主会话里，子 Agent 直接调用 `EnterPlanMode` 会被拒绝。第 9 章写明，这是用户级决策，子 Agent 没有面向用户的审批通道，放进去会永远卡在等待批准的状态。多 Agent 或 teammate 场景则把退出改写成发给领导的 `plan_approval_request`，检查点还在，只是「人」被替换成了「领导」。

还有一层断路器。如果进入前是 auto 一类的自动审批，而规划期间自动审批的闸门已经熔断，退出时不能盲目恢复到 auto，否则「进一下再出」本身就成了绕过熔断的通道。Bazingga 和第 9 章都单独写了这个分支：目标状态不合法时，降级回到 default。

退出还可以顺便做一次语义级的预授权。s10 写到，`ExitPlanMode` 支持 `allowedPrompts`：目前的 schema 只接受 `Bash`，附带一段语义描述，例如 `run tests`、`npm install`。用户批准计划的同时，也就批准了这组后续操作，实施阶段就不用反复弹窗。

这套硬约束也有明确的失效条件。Tim 写得很直白：如果会话是带着 `--dangerously-skip-permissions` 启动的，再切进计划模式，你买到的往往只是一句措辞强硬的建议——模型仍被要求先规划、不改文件，但一次写入可能会直接通过。计划模式不是魔法，它站在权限模型上面；会话一开始就绕开了权限，这层信封也就不再是硬的。

工作流本身也不是只有一种。第 9 章写了两套，用 feature gate 切换。默认是五阶段：先用 Explore 子 Agent 摸清代码，再用 Plan 子 Agent 做设计，然后审阅、把最终方案写入计划文件，最后调用 `ExitPlanMode`。HP Frei 把这两套机制拆得很清楚：计划模式是外层的权限信封，Plan 子 Agent 是信封内部的研究工人，两者不是互相替代的关系。

另一套是访谈式的迭代循环：先快速扫几个关键文件，写出骨架计划就向用户提问，而不是穷尽探索后再开口。两边都有一条相同的纪律：不能用普通文本去问「这个计划可以吗」，那句话必须走 `ExitPlanMode` 或专门的提问工具，审批通道只放行可执行的承诺。

生产数据也在反过来改计划的写法。第 9 章记录了一组以 plan-exit 计、覆盖 14 天、样本量 N=26.3M 的基线：计划文件 p50 为 4,906 字符，p90 为 11,617 字符，均值 6,207 字符。拒绝率与长度正相关——不到 2K 的计划拒绝率约 20%，超过 20K 的高达 50%。团队为此做了名为 `tengu_pewter_ledger` 的 A/B，在 control、trim、cut、cap 四种格式之间试更短的计划是否能提高满意度。计划写得越长，并不等于用户越买账。

## 同一目的，不同接法

不是每家都把计划模式做成全局权限状态。

[Cline 更早把 Plan/Act 做成一等公民](https://cline.ghost.io/plan-smarter-code-faster-clines-plan-act-is-the-paradigm-for-agentic-coding/)。官方博客强调的是先把上下文和策略对齐，再放手执行，而且计划和实施可以选用不同的模型。

[AIGC Camp 对 2026 年中源码的剖析](https://aigccamp.com/source/cline)则指出，Cline 的切换不是改一个全局 PermissionMode，而是用两个特殊工具：`plan_mode_respond` 和 `act_mode_respond`。Plan 模式下只允许前者加上只读工具，Act 模式下只允许后者加上写工具；用户在界面里切换模式，就是在换可用工具集。它用工具把当前模式暴露给模型，而不必把 mode 做成另一份全局状态。

代价也很具体。模型常常写到一半又想再读文件，于是 `plan_mode_respond` 后来加了 `needs_more_exploration`；YOLO 模式下写完方案可以不经审批自动切到 Act——闸门一旦打开，这层对齐也就不再硬。

Cursor 走的是另一条产品判断。[2025 年 10 月的 Plan Mode 介绍](https://cursor.com/blog/plan-mode)给了模型创建和更新计划的工具，以及可以直接改计划的编辑器；[2026 年 1 月 30 日的后续文章](https://cursor.com/blog/dynamic-planning-agents)把重点放在「计划要活着」。它明确反对把计划冻成一份过早锁死的规格：Agent 一边执行一边勾任务，发现约束就改计划，跑偏了就回到计划、撤回改动、改完再跑一遍。计划可以存成工作区里受版本控制的 `.cursor/plans/` 文件，跟代码放在一起。

和 Claude Code「批准后退出、恢复写权限、按这份计划实施」相比，Cursor 更强调执行过程中计划仍然是可编辑的循环表面，而不是一次性的门禁。

Windsurf 的 [Wave 10 Planning Mode](https://devin.ai/blog/windsurf-wave-10-planning-mode)（2025 年 6 月 10 日）把「活着的计划」又拆成两条时间线。前台是用户选中的模型，按计划做短期动作；后台是一个更擅长长期推理的更大模型，持续迭代那份本地 Markdown 计划。人和 Agent 都可以改这份文件，计划变了，短动作也要跟着变。Cognition 的说法是：缺的不是聊天界面，而是一个能把长期计划时间线编码进去的原语；Markdown 文件是他们给这个原语的 v1。

这几条路并不互相否定。[loams.dev 对多家 harness 的拆解](https://www.loams.dev/blog/coding-agent-harnesses)把计划模式看成权限梯子上最谨慎的一档：先进入只读阶段，再允许编辑。[Kieran Potts 对 agent harness 的整理](https://kieranpotts.com/garden/agent-harness.html)也把 plan 写成一种捆绑：一边是自动放行读、挡住写的权限模式，一边是注入到提示词里的行为说明。模式不只是一把锁，也是在告诉模型：现在该调研、提问和提案，而不是去撞那扇门。

开源 harness 已经在抄这套语义。 2026 年 7 月，[go-code 的 #819](https://github.com/dennisonbertram/go-code/issues/819) 把目标写得很清楚：计划期间只能写指定的 `.harness/plan.md`，退出必须停下来等人批准，即使 `full_auto` 也不能绕过。这和 Tim 观察到的 Claude Code 在 `--dangerously-skip-permissions` 下可能失去硬拦截，恰好是相反的取舍——有人把「退出审批不可被 YOLO 绕过」当成要钉死的不变量。

## 对自己做 Agent 的人

如果只从这些文章里带走一条工程判断，那就是：不要把计划模式做成一句「请先规划」的 system prompt。

乔人尹给 mini-agent 留的练习已经点到了最小骨架——加一种只放行只读工具的权限模式，再加进入和退出的命令——但做成可靠，还得把计划写成文件，把退出做成审批，并接受软约束会漂移这个前提。能用电闸解决的，不必只靠告示。

不同产品在信封怎么封、计划要不要在执行中继续改、要不要把规划模型和执行模型拆开，取舍并不相同。共同点是：对齐发生在最便宜的时刻，并且这份对齐有一个不依赖模型自觉的落点。

## 参考文献

- [What Actually Is Claude Code’s Plan Mode?](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/)，Armin Ronacher，2025 年 12 月 17 日
- [Claude Code核心拆解：计划模式——怎么让模型「先对齐再动手」](https://jishuzhan.net/article/2121098887466057730)，Bazingga，技术栈，2026 年 11 月 13 日
- [s10 — Plan 模式：先想后做](https://mo-jiao.github.io/claude-code-source-code/guide/s10-plan-mode.html)，mo-jiao，Claude Code 源码解读
- [第 9 章：Plan 模式](https://github.com/Windy3f3f3f3f/how-claude-code-works/blob/main/docs/10-plan-mode.md)，Windy3f3f3f3f，how-claude-code-works
- [Claude Code 解剖，对照自己的实现](https://underestimated.cn/archives/claude-code-anatomy.html)，乔人尹，2026 年 10 月 10 日
- [Claude Code's Two Planning Mechanisms: Plan Mode vs Plan Subagent](https://hpfreilabs.com/blog/claude-code-planning-mode-vs-planning)，HP Frei Labs，2026 年 5 月 21 日
- [Claude Code plan mode: decide before the agent writes](https://tim-schipper.nl/en/blog/claude-code-plan-mode)，Tim Schipper，2026 年 8 月 2 日
- [Cline 源码剖析](https://aigccamp.com/source/cline)，AIGC Camp
- [Plan Smarter, Code Faster: Cline's Plan & Act](https://cline.ghost.io/plan-smarter-code-faster-clines-plan-act-is-the-paradigm-for-agentic-coding/)，Nick Baumann，Cline，2025 年 5 月 6 日
- [Introducing Plan Mode](https://cursor.com/blog/plan-mode)，Jai Smith，Cursor，2025 年 10 月 7 日
- [Dynamic planning with agents](https://cursor.com/blog/dynamic-planning-agents)，Denzil Bernard、Diggory Rycroft，Cursor，2026 年 1 月 30 日
- [Wave 10: Planning Mode](https://devin.ai/blog/windsurf-wave-10-planning-mode)，Cognition，2025 年 6 月 10 日
- [Coding-agent harnesses, taken apart](https://www.loams.dev/blog/coding-agent-harnesses)，Dinakaran，Loams，2026 年 10 月 18 日
- [Agent harness](https://kieranpotts.com/garden/agent-harness.html)，Kieran Potts
- [epic(harness): complete plan mode](https://github.com/dennisonbertram/go-code/issues/819)，dennisonbertram，go-code，2026 年 7 月 19 日

## License
* 本文遵守创作共享 CC BY-NC-SA 3.0协议
