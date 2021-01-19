---
layout: post
title: "[译文]Implementing Raft: Part 1 - Elections"
subtitle: "Implementing Raft"
date: 2021-01-18
author: "ChenJY"
header-img: "img/ostep.png"
catalog: true
tags: 
    - 分布式系统
    - 论文阅读
    - 算法
    - 共识协议
    - Golang
---

> 这一系列文章翻译自 Eli Bendersky 的博客，原文地址：https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/，翻译已获得原文作者许可，禁止转载和商用

这是讲解 Raft 一致性协议和 Go 语言实现系列文章的第二篇，完整的目录如下：

- Part 0: Introduction
- Part 1: Elections (this post)
- Part 2: Commands and log replication
- Part 3: Persistence and optimizations

在本文中，我会尝试讲解 Raft 代码的总体结构，主要聚焦于 leader election 部分的源码实现。这部分源码附带了完备的功能测试，你可以手动运行它们，并观察系统的执行情况。当然目前该项目还不能响应客户端请求，也没有完成Raft Log 持久化，这些部分将在第三篇文章中讲到。

### Code structure

我们简单讲解一下 Raft 的代码是如何组织的，这一讲解适用于这一系列的所有文章。

通常，Raft 库会被实现成一个依赖包，你可以将它嵌入一些服务（service）中使用。由于我们并不是真正开发一个服务，而只是学习实现 Raft 算法本身，因此我创建了一个 Server，它里面包含了一个共识模块（CM），Server 整体通过 RPC 与外部通信。通过这种抽象方式，我将关键的代码实现都剥离了出来，方便你阅读。

![](https://eli.thegreenplace.net/images/2020/server-cm-architecture.png)

一致性模块（CM）是 Raft 算法的核心，它位于 raft.go 文件中。它的实现完全不涉及网络、如何与集群中的其他副本的连接等，CM 中唯一跟网络有关系的部分是：

```java
// id is the server ID of this CM.
id int
// peerIds lists the IDs of our peers in the cluster.
peerIds []int
// server is the server containing this CM. It's used to issue RPC calls
// to peers.
server *Server
```

在 Raft 的实现中，每一个 Raft 副本把集群中其他副本称作 “peers”。每一个 peer 在集群中都有一个唯一 ID，并维护一个保存所有 peers 的列表。属性 server 是一个指向 Server 结构体的指针（实现代码在 server.go 中），它的作用是让 ConsensusModule 能够向 peers 发送消息。我们下文会看到它具体是如何工作的。

这样设计的目的，是为了屏蔽掉所有网络细节，只专注于 Raft 算法实现。如果你想要对照 Raft 原始论文理解这个项目，那你只需要关注 ConsensusModule 与它内部的方法。Server 的代码是一个相当简单的 Go 网络脚手架，为了撰写单元测试，我在其中可能略微夹杂了一些逻辑。这一系列文章中，我不会花很多时间叙述 Server 的代码，但如果你对它有任何疑问，可以直接提出。

### Raft server states

从上帝视角看，Raft CM 是一个由三个状态（state）组成的状态机模型：

![](https://eli.thegreenplace.net/images/2020/raft-highlevel-state-machine.png)

由于第一篇文章我们说过 Raft 是用来实现状态机的，那么读到这里你可能有点疑惑—— “Raft 自己也有状态机？”。其实，state 这个术语在这里需要被重新定义。Raft 确实是一个被用来实现任意复制状态机的算法，但它内部确实也有一个状态机。随着文章深入，你会发现，Raft 算法中一个 server 究竟处于何种 state 需要非常明确。

当一个 Raft 集群中所有 server 的 state 都稳定时，其中一个 server 是 leader，其他的 server 都是 follower。虽然我们期望这种状态能永远正常运行保持不变，但很可惜事实不会如我们所愿，好在 Raft 被设计出来的目的就是为了容错（fault tolerant），因此接下来我们会讨论一些异常场景，例如一些 server crash 或者断连等。

就如之前文章提及的那样，Raft 使用了一个很严格的 leadership 模型。Leader 负责处理客户端请求，向 Raft log 新增记录，并把新记录复制给其他 follower。每个 follower 都在时刻准备着，一旦发生 leader crash 或者失联时接管领导权成为新 leader。这就是指上图中 “times out，start election” 描述的状态转移变化。

### Terms

如同现实生活中普通的领导人选举一样，Raft 里也有 leader 任期的概念。任期代表一段时间，这段时间内某个固定的 server 一直是 leader。每一次新的选举都会触发一个新的任期，Raft 保证了一个任期内仅有一个 leader。

这种类比不适合继续深入，因为毕竟 Raft 的选举与现实生活中的选举很不一样。Raft 中的选举过程更强调协作性；Raft 中的候选节点（candidates）的目的并非不择手段保证自己赢得选举——而是确保让合适的 server 赢得选举，开始新的任期。我们很快就会详细讨论，什么是“选择合适的 server”。

### Election timer

构建 Raft 算法的关键就是选举计时器。每个 follower 都会一直运行这个计时器，每当 follower 从 leader 收到心跳后，它就会重置计时器。Leader 会向 follower 发送定时心跳，所以当一个 follower 长时间收不到心跳时就意味着 leader crash 了或者自己与 leader 断连了，它会在计时器结束时发起一轮新的选举（自身 state 从 follower 转为 candidate）。

Q：有没有可能，同一时间所有的 follower 都转变为 candidate？

A：选举计时器的时间是随机的，这也是 Raft 能保持算法简洁的关键。Raft 通过这种随机时间尽可能降低大多数 follower 同时开启选举的可能性。但即使它们同时变为 candidate，在一个任期内也只有一个 candidate 可以当选为 leader。如果出现某一次选举中，没有节点得到大多数选票（majority）的话，那么新一轮选举（term 当然也要更新）会重新发起，这种场景比较罕见。理论上存在无限期不停选举的可能性，但随着一轮一轮选举，这种可能性是越来越低的。

Q：如果一个 follower 和集群间的其他机器发生网络分区了怎么办？它不是会自己单独发起一轮新选举嘛？因为无法从 leader 获取心跳了。

A：这就是网络分区的阴险之处，因为 follower 无法意识到，到底是自己被隔离了，还是 leader 被隔离了。因此它确实会在计时器结束时发起一轮新的选举，但是由于它无法和大多数节点通信，因此无法收到大多数选票，不可能当选为 leader。它就会一直停留在 candidate 状态不停自旋，直到网络分区恢复，它重新连上集群。我们后文会更深入讨论这种场景。

### Inter-peer RPCs

peers 互相之间仅发送两类 RPC 命令，想要了解关于这两类 RPC 命令的细节参数和规则，可以查阅论文的 Figure2。这里我仅简要描述二者的作用：

- RequestVotes（RV）：仅在 candidate 状态使用；candidates 进行选举时，使用此命令向其他 peers 收集选票。其他 peers 会返回“是否将票投给你”这个关键信息。
- AppendEntries（AE）：仅在 leader 状态下使用；leader 使用此命令来复制 Log Entry 给 followers，但此命令也被复用来发送心跳。这个 PRC 命令会周期性的发送给所有 follower，即使参数内没有任何新 Log Entry 需要复制。

聪明的读者会发现，follower 不需要发送任何 RPC 命令。这是正确的；follower 不需要互相发送 RPC，但它们需要在后台运行选举计时器。如果在计时期间内未能从 leader 收到心跳，则 follower 转变为 candidate 并开始发送 RV 请求，触发新一轮选举。

### Implementing the election timer

是时候深入源码了。下面所有的代码都位于这个文件中，我不会展示所有的 fields，如果你感兴趣可以直接看源码文件。
CM 通过下面的函数，在一个 goroutine 中运行选举计时器：

```java
func (cm *ConsensusModule) runElectionTimer() {
  timeoutDuration := cm.electionTimeout()
  cm.mu.Lock()
  termStarted := cm.currentTerm
  cm.mu.Unlock()
  cm.dlog("election timer started (%v), term=%d", timeoutDuration, termStarted)
  // This loops until either:
  // - we discover the election timer is no longer needed, or
  // - the election timer expires and this CM becomes a candidate
  // In a follower, this typically keeps running in the background for the
  // duration of the CM's lifetime.
  ticker := time.NewTicker(10 * time.Millisecond)
  defer ticker.Stop()
  for {
    <-ticker.C
    cm.mu.Lock()
    if cm.state != Candidate && cm.state != Follower {
      cm.dlog("in election timer state=%s, bailing out", cm.state)
      cm.mu.Unlock()
      return
    }
    if termStarted != cm.currentTerm {
      cm.dlog("in election timer term changed from %d to %d, bailing out", termStarted, cm.currentTerm)
      cm.mu.Unlock()
      return
    }
    // Start an election if we haven't heard from a leader or haven't voted for
    // someone for the duration of the timeout.
    if elapsed := time.Since(cm.electionResetEvent); elapsed >= timeoutDuration {
      cm.startElection()
      cm.mu.Unlock()
      return
    }
    cm.mu.Unlock()
  }
}
```

一开始，它会通过 cm.electionTimeout 方法选择一个随机的倒计时时间。这里我们使用的随机范围是 150~300ms，同论文中提及的一致。如同大多数 ConsensusModule 中的方法一样，runElectionTimer 通过加锁保证对属性的原子访问。这很重要，可以让我们的实现尽可能保证线程安全。这意味着顺序书写的代码也是顺序执行的，不会在多个 event handler 中被错乱执行。RPC 命令仍然是并发的，正因如此我们更需要保护可能被共享的数据结构。我们很快就会阅读到 RPC handler 的实现。

这个方法中的主循坏每隔 10ms 运行一次计时器。当然，还有更高效的方法可以用来等待和处理时间事件，但是这种写法最简单。循环每经过 10ms 迭代执行一次，理论上我们当然可以让这个循环每次间隔一个计时器超时周期运行一次，但这样的缺点是 follower 的响应速度会变慢，并且难以 debug。我们每次都会检查 state 是否是期望的 state，且 term 是否未发生改变，如果任意一个发生了变化，我们就会终止选举倒计时器。

如果自上一个重置计时器的事件（election reset event）之后，过了足够长的时间（即计时结束），那么这个 follower 就会发起一轮新选举，自身状态变为 candidate。什么事件可以用来重置计时呢？答案是：任何一个可以终止选举的事件，例如，收到一个合法的新心跳，或者投票给其他 candidate。我们很快会阅读到这块的代码。

### Becoming a candidate

上面已经提到，当长时间未能从 leader 处收到心跳后，follower 就会开启选举。在我们阅读代码之前，让我们思考下，发起选举我们需要哪些东西？

1. 将自身状态转变为 candidate 并增加 term，因为这是算法明确要求每次开启选举之前要做的事情。
2. 向所有 peers 发送 RV 请求，请它们在这轮选举中给自己投票。
3. 等待这些 RPC 的响应结果，检查得票数看看自己能否当选 leader。

使用 Go 的话所有这些逻辑都可以在一个函数内实现：

```java
func (cm *ConsensusModule) startElection() {
  cm.state = Candidate
  cm.currentTerm += 1
  savedCurrentTerm := cm.currentTerm
  cm.electionResetEvent = time.Now()
  cm.votedFor = cm.id
  cm.dlog("becomes Candidate (currentTerm=%d); log=%v", savedCurrentTerm, cm.log)
  var votesReceived int32 = 1
  // Send RequestVote RPCs to all other servers concurrently.
  for _, peerId := range cm.peerIds {
    go func(peerId int) {
      args := RequestVoteArgs{
        Term:        savedCurrentTerm,
        CandidateId: cm.id,
      }
      var reply RequestVoteReply
      cm.dlog("sending RequestVote to %d: %+v", peerId, args)
      if err := cm.server.Call(peerId, "ConsensusModule.RequestVote", args, &reply); err == nil {
        cm.mu.Lock()
        defer cm.mu.Unlock()
        cm.dlog("received RequestVoteReply %+v", reply)
        if cm.state != Candidate {
          cm.dlog("while waiting for reply, state = %v", cm.state)
          return
        }
        if reply.Term > savedCurrentTerm {
          cm.dlog("term out of date in RequestVoteReply")
          cm.becomeFollower(reply.Term)
          return
        } else if reply.Term == savedCurrentTerm {
          if reply.VoteGranted {
            votes := int(atomic.AddInt32(&votesReceived, 1))
            if votes*2 > len(cm.peerIds)+1 {
              // Won the election!
              cm.dlog("wins election with %d votes", votes)
              cm.startLeader()
              return
            }
          }
        }
      }
    }(peerId)
  }
  // Run another election timer, in case this election is not successful.
  go cm.runElectionTimer()
}
```

candidate 最开始先投票给自己——通过初始化 votesReceived 为 1 并赋值 cm.votedFor = cm.id。
然后它并行发送 RPC 请求给所有的 peers。每个 RPC 请求都运行在一个新开的 goroutine 中，这是提高选举效率的方式。下面展示了 RV 这类 RPC 请求是如何调用的：

```java
cm.server.Call(peer, "ConsensusModule.RequestVote", args, &reply)
```

我们使用 ConsensusModule.server 属性指向的 Server 发起远程调用，ConsensusModule.RequestVotes 是远程方法的名字，这行代码最终会调用第一个参数 peer 指代的机器的 RequestVote 方法。

如果 RPC 调用成功，那么由于又过了一小段时间（RPC 调用耗时），我们需要检查 state 来确认刚刚那一小段时间内，自己的角色是否发生了变化，如果我们的角色已经不再是 candidate 的话，直接退出选举流程。什么时候会发生这种情况呢？例如我们可能已经从其他 RPC 请求的计算结果中收到了足够的选票，从而当选成功 leader。或者我们接收到了从一个拥有更新的任期的 leader 发来的心跳，因而决定将自身的状态转变为 follower。我们要记住，网络状况往往是不可预测的，因此有可能一个 PRC 请求会阻塞很长时间——在此期间内，Raft 算法其他的判断逻辑可能已经向前走了一大步了，因此我们要做的就是及时判断自身的 state 是否被更新，必要时立即放弃正在进行的选举，追随大多数节点的选择，保证 Raft 算法尽快顺利执行下去。

如果接收到响应结果时，我们自身的角色还是 candidate 的话，我们检查响应结果里的 term 是否和我们发出 RV 请求时的原始 term 值相同，如果响应结果里的 term 更新，我们就要立即转变角色成为 follower。这种情况常往往发生在：我们还在收集选票时，已经有其他节点成功当选 leader 了。

如果响应结果里返回的 term 跟我们发送 RV 时原始的 term 相同，继续检查对方是否同意投票给自己。我们使用了一个原子变量 votes 来在多个 goroutine 的情况下原子性计算选票数目，保证线程安全。如果当下的 server 已经获得大多数选票，它就当选为 leader。

注意，startElection 方法并不是阻塞的。它更新一些 state，运行一组 goroutine 之后返回。因此，它需要在最后一行代码处再用 goroutine 启动一个新的选举计时器。这种方式保证了，如果这轮选举中未达成任何有效的结论，这个节点在倒计时结束后会再开启一轮新的选择。这行代码也解释了，为什么 runElectionTimer 方法中要有 state 检查逻辑：如果这轮选举确实让当下这个节点成为了 leader，那么当并发运行的 runElectionTimer 方法观察到 state 改变时，会立即 return。

### Becoming a leader

上面的代码中，我们看到，如果选票达标的话，startElection 方法会调用 startLeader。后者的代码如下：

```java
func (cm *ConsensusModule) startLeader() {
  cm.state = Leader
  cm.dlog("becomes Leader; term=%d, log=%v", cm.currentTerm, cm.log)
  go func() {
    ticker := time.NewTicker(50 * time.Millisecond)
    defer ticker.Stop()
    // Send periodic heartbeats, as long as still leader.
    for {
      cm.leaderSendHeartbeats()
      <-ticker.C
      cm.mu.Lock()
      if cm.state != Leader {
        cm.mu.Unlock()
        return
      }
      cm.mu.Unlock()
    }
  }()
}
```

这其实是个很简单的方法：它所做的事情就是开启一个心跳定时器——一个调用 leaderSendHeartbeats 方法的 goroutine，只要 CM 还是 leader 时它就会每隔 50ms 运行一次。leaderSendHeartbeats 方法体如下：

```java
func (cm *ConsensusModule) leaderSendHeartbeats() {
  cm.mu.Lock()
  savedCurrentTerm := cm.currentTerm
  cm.mu.Unlock()
  for _, peerId := range cm.peerIds {
    args := AppendEntriesArgs{
      Term:     savedCurrentTerm,
      LeaderId: cm.id,
    }
    go func(peerId int) {
      cm.dlog("sending AppendEntries to %v: ni=%d, args=%+v", peerId, 0, args)
      var reply AppendEntriesReply
      if err := cm.server.Call(peerId, "ConsensusModule.AppendEntries", args, &reply); err == nil {
        cm.mu.Lock()
        defer cm.mu.Unlock()
        if reply.Term > savedCurrentTerm {
          cm.dlog("term out of date in heartbeat reply")
          cm.becomeFollower(reply.Term)
          return
        }
      }
    }(peerId)
  }
}
```

它跟 startElection 的代码很相似，基本上只替换了 RPC 请求，它通过运行一组 goroutine 向每个 peer 发送心跳，这里的心跳请求 RPC 中不带有任何 Log Entry 信息，这种 AE RPC 类型在 Raft 算法中仅作为心跳使用。
跟处理 RV RPC 的响应相似，如果请求结果中包含了更新的 term 则接收到的这个 peer 立刻将自身角色转变为 follower，现在，是时候看看 becomeFollower 方法了：

```java
func (cm *ConsensusModule) becomeFollower(term int) {
  cm.dlog("becomes Follower with term=%d; log=%v", term, cm.log)
  cm.state = Follower
  cm.currentTerm = term
  cm.votedFor = -1
  cm.electionResetEvent = time.Now()
  go cm.runElectionTimer()
}
```

它设置 CM 的状态为 follower 并且重置了 term 和其他关键的 state 属性。它也开启了一个新的选举倒计时器，因为身为 follower 他就是应该在后台（background）不停运行这个计时器的，一旦计时结束就要转变为 candidate 开始发起选举。

### Answering RPCs

目前为止，我们看到的都是 Raft 算法实现中主动发起的部分——例如初始化 RPC 请求、计时器、状态流转。现在我们要看一下其他 peers 如何处理这些远程 RPC 调用的。我们从如何响应 RequestVote 请求开始：

```java
func (cm *ConsensusModule) RequestVote(args RequestVoteArgs, reply *RequestVoteReply) error {
  cm.mu.Lock()
  defer cm.mu.Unlock()
  if cm.state == Dead {
    return nil
  }
  cm.dlog("RequestVote: %+v [currentTerm=%d, votedFor=%d]", args, cm.currentTerm, cm.votedFor)
  if args.Term > cm.currentTerm {
    cm.dlog("... term out of date in RequestVote")
    cm.becomeFollower(args.Term)
  }
  if cm.currentTerm == args.Term &&
    (cm.votedFor == -1 || cm.votedFor == args.CandidateId) {
    reply.VoteGranted = true
    cm.votedFor = args.CandidateId
    cm.electionResetEvent = time.Now()
  } else {
    reply.VoteGranted = false
  }
  reply.Term = cm.currentTerm
  cm.dlog("... RequestVote reply: %+v", reply)
  return nil
}
```

你或许注意到了，代码中对 “dead” 这个 state 的检查逻辑，我们后续再讲这个。

方法开头的逻辑很眼熟，也是检查 term 如果一旦过期就转变为 follower。如果它已经是 follower，则 state 不变但是要重置其他属性。

否则，如果调用方（caller）的 term 与我们一致并且我们还未投票给任何 candidate，则我们将票投给这个 caller。注意，我们绝不响应任何包含过期 term 的选举请求。

下面是 AppendEntries 方法的代码：

```java
func (cm *ConsensusModule) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) error {
  cm.mu.Lock()
  defer cm.mu.Unlock()
  if cm.state == Dead {
    return nil
  }
  cm.dlog("AppendEntries: %+v", args)
  if args.Term > cm.currentTerm {
    cm.dlog("... term out of date in AppendEntries")
    cm.becomeFollower(args.Term)
  }
  reply.Success = false
  if args.Term == cm.currentTerm {
    if cm.state != Follower {
      cm.becomeFollower(args.Term)
    }
    cm.electionResetEvent = time.Now()
    reply.Success = true
  }
  reply.Term = cm.currentTerm
  cm.dlog("AppendEntries reply: %+v", *reply)
  return nil
}
```

这块逻辑跟论文中的 Figure2 也是对应的，不做赘述。我们仅需要注意这样一个 tricky 的判断条件：

```java
if cm.state != Follower {
  cm.becomeFollower(args.Term)
}
```

Q：为什么要保证 term 相同时它（可能是个 candidate）立即成为其他 leader 的 follower 呢？

A：Raft 保证了一个 term 中仅有一个 leader，如果你仔细阅读 RequestVote 和 startElection 中发送 RV 请求的代码，就会发现不可能允许相同的任期内有两个 leader 。这个判断条件很关键，它保证了 candidates 能感知到其他 peer 在当前的 term 下赢得了选举。

### States and Goroutines

我们需要给 CM 所有可能的 state 和 CM 可能运行的所有 goroutine 做一个总结。

- Follower：当 CM 是 follower 时，每一次对 becomeFollower 方法的调用，都会诞生一个新的 goroutine 用来运行 runElectionTimer。注意，可能会出现短时间内多个 goroutine 同时在运行 runElectionTimer 方法。例如一个 follower 从一个 leader 收到了 term 更新的 RV 请求；此时 becomeFollower 方法会立即触发一个新的 goroutine。但是，当老的 goroutine 检测到 term 变更后，它会及时退出。
- Candidate：一个 candidate 也会同时运行多个 election goroutine，且它还有一组用来发送 RPC 请求的 goroutines。它同 follower 一样，也保证了当新的 goroutine 运行起来之后，老的会及时退出。如果等到响应结束再退出那些 RPC goroutine 会花很长的时间，因此最好的方式是，当他们意识到自己已经过时的那一刻立即默默退出。
- Leader：leader 是没有 election goroutine 的，但它有 heartbeat goroutine，每隔 50ms 运行一次。
代码中还有一个单独的 state ——Dead。它用来保证 CM 有序 shutdown。Stop 请求会将 CM 的 state 置位 Dead 然后所有的 goroutine 检查到这个 state 后都会退出。

这么多 goroutine，万一代码有 bug 让所有这些 goroutine 都运行起来的话，可能令人感到担心和不可控。万一其中一些持续在后台运行；或者更糟的是，他们没有正常退出数目还一直增长的话怎么办？这就是 leak check 要做的事情，我写了若干个检测泄露情况的单元测试。这些单元测试无序地运行 election 并保证最后不存在零星的 goroutine 没有退出（调用 Stop 请求后等待一段时间让所有的 goroutine 都能退出）。

### Runaway server and increasing terms

总结本文前，让我们再了解 Raft 里一个 tricky 的场景，看看 Raft 是如何处理的。这个例子非常有趣，并且有教育意义。我尝试通过讲故事的方式呈现它，你可能也需要拿一张纸跟着我的故事记录下不同 server 所处的 state。如果你跟不上这个例子——请发邮件给我，我会想办法把它讲得更通俗易懂一些。

假设一个三机集群：A，B 和 C，其中 A 是 leader，初始时的 term 是 1 整个集群正常运行。A 会每隔 50ms 发送心跳给 B 和 C，并正常收到响应。每一个 AE 请求都会重置 B 和 C 的选举计时器，正常情况下他们会一直处于 follower 状态。

某个时刻，由于 server B 的网络交换机出了个小问题，server B 跟 AC 之间发生了网络分区。A 照常每隔 50ms 发送心跳，但给 B 的心跳请求要么立即返回异常，要么延时很久后超时了。针对这种情况，A 无法做什么操作，集群问题也不大。虽然我们还没讲到 Log 复制，但是只要知道此时三机集群中两个 server 保持正常，这个集群依旧可以处理客户端请求。

那 B 怎么办呢？假设 B 的选举计时器的超时时间是 200ms，当发生网络分区大约 200ms 后，B 的 runElectionTimer goroutine 意识到自己已经很久没有从 leader 收到心跳了，B 没有任何办法知道到底是自己出了异常还是 leader 出了异常，因此它能做的只是转变为 candidate 并发起选举。

B 的 term 会自增成 2（此时 A 和 C 的 term 还是 1）。B 会尽职地向 A 和 C 发送 RV 请求恳请他们投票给自己；B 会一边开启新的选举倒计时器，随机选择一个 150~300ms 之间的超时时间，例如 250ms，一边继续等待看看之前的选举到底有没有产生结果。但很显然，这些 RPC 由于分区无法达到 A 和 C，所以选举没有具体的结论，B 会在 250ms 结束时再次发起新的选举，此时 B 的 term 再次自增到 3。

持续反复，B 的交换机若干秒后恢复正常，B 与 AC 之前的网络分区结束，同时由于这期间 B 不停地发起选举，它的 term 已经自增到 8 了。

这时候，B 重新与集群连通，A 发送的定期心跳请求再次到达 B。B 的 AppendEntries 方法被调用，并且向 A 返回了自身更新的 term = 8。

A 在 leaderSendHeartbeats 方法中收到了 B 返回的响应，发现 B 的 term 更新，A 马上更新自己的 term 为 8 并且转变为 follower。整个集群暂时丢失了 leader。

现在，由于时间导致的不确定性，这个集群后续会出现很多种可能性。B 是一个 candidate，但它可能在网络分区恢复之前刚刚发送 RVs；C 是一个 follower，但由于没有从 A 收到 AEs 心跳，因此当它自己的倒计时结束时会转变为 candidate。A 成了一个 follower，并且当倒计时结束后会迅速开始选举。

所以上面三个节点中每个都可能赢得选举。注意，发生这种情况的原因只是由于我们还没有开始复制任何 Log。我们在接下去的文章中会讲到，在实际的场景中，即使 B 发生了分区，但 A 与 C 还会正常处理客户端的请求，因此他俩的 Log 会比 B 更新，B 不可能成为 leader——只有 A 和 C 才可能赢得选举；我们会在后文中回顾这个场景。

假设自 B 分区后没有任何新的命令抵达集群，那么当 B 重连之后即使发生 leader 变更也是合理且正常的。

也许你会发现，这个场景（如果没有任何命令抵达集群）下其实根本没必要发生 leader 变更，这种 leader 变更反而使算法变得低效了，因为 A 作为 leader 一直在正常运行。但是通过牺牲一些效率换来算法设计的简洁是 Raft 追求的，毕竟集群大多数时间（99.9%）都是正常运行的，发生这种极端分区的可能性很小。

### What's Next

为了确保你学会了如何实现 Raft 而不是仅仅停留在理论理解上，我强烈建议你上手运行一下代码中的测试。

仓库的 README 文件有更详细的指导，教你如何运行测试、观察结果。代码附带了很详尽的单元测试，涉及了许多特殊的场景（包括我们上述提及的），运行测试观察日志输出有益于你更好地理解代码实现。注意到 cm.dlog(...) 这个方法了吧。仓库中提供了一个小工具，它可以将日志聚合起来显示在 HTML 上，如果想了解更多请阅读 README。动手跑一些测试，观察日志输出，如果想更细致地钻研细节，你可以自己添加 dlog。

第三篇文章将叙述一个更完整的 Raft 实现，它可以处理客户端命令并且在集群中复制这些命令。敬请关注！
