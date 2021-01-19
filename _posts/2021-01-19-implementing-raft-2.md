---
layout: post
title: "[译文]Implementing Raft: Part 2 - Commands and Log Replication"
subtitle: "Implementing Raft"
date: 2021-01-19
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

> 这一系列文章翻译自 Eli Bendersky 的博客，原文地址：https://eli.thegreenplace.net/2020/implementing-raft-part-2-commands-and-log-replication/，翻译已获得原文作者许可，禁止转载和商用

这是讲解 Raft 一致性协议和 Go 语言实现系列文章的第二篇，完整的目录如下：

- Part 0: Introduction
- Part 1: Elections
- Part 2: Commands and log replication (this post)
- Part 3: Persistence and optimizations

本文中，我们会进一步丰富之前的 Raft 实现，让它能真正处理客户端提交的请求，并让数据在 Raft 集群间完成复制。项目的代码结构与上一篇中一样没有变化，但新增了一些结构体、方法——下文我会提到。

本文涉及到的代码都在这个[目录](https://github.com/eliben/raft/tree/master/part2)中。

### Client interaction

在第一篇文章中我们简要讲过客户端交互流程（client interaction）；我强烈建议你回过头再看下那部分内容。这里我们着重讲解客户端如何找到 leader，然后我们讲解当它找到 leader 后会发生什么。

之前讨论过，客户端使用 Raft 协议复制一连串的指令流（commands），这一连串指令流可以被认为是给予状态机的输入。就我们的 Raft 实现而言，这些指令的内容是完全随意的，我们使用 Go 语言中的 interface{} 结构指代（译者注，类似于 Java 中的 Object）。一个指令在 Raft 中会经历下面的过程：

1. 首先，客户端将指令发送（submit）给 leader，也就是说，在一个 Raft 集群中这么多 peers 里，一个指令只会发送给其中一个 peer。
2. leader 负责将指令复制给其他 followers。
3. 最后，当 leader 确认指令已经复制到多数派的 peers 后（即 a majority of peers 都成功响应：他们已经将该指令存入自己的 log 中）这个指令就被提交（commit）了，leader 再向客户端返回结果。

注意下发送（submit）指令和提交（commit）指令的区别——由于下文我们要讲解这部分的实现，因此谨记这二者的区别非常重要。一个指令发送给一个单独的 Raft peer 节点，但一段时间过后，多数派的 peers （更确切的说是所有连接正常且存活的 peers）都应该 commit 这个指令并通知他们自己的 “client”（译者注：这个 client 不同于之前讲过的 submit 命令的外部客户端，而是指的连接 CM 的状态机，你可以类比成一个 kv 数据库）。

回顾下第一篇文章中的图：

![](https://eli.thegreenplace.net/images/2020/raft-consensus-module-log.png)

三个小蓝点组成的状态机代表了任意一个使用 Raft 做数据复制的服务；例如，可以是一个 kv 数据库。被提交（commit）的指令会更改这个服务的内部状态（例如往数据库中新添加一个 kv 键值对）。

然后我们所说的 “client” 其实是在 Raft CM 上下文中指代的客户端，它通常指的就是状态机服务，因为这是指令被提交之后最终被应用的地方。换句话说，就是上图中 CM 跟状态机之间的黑色箭头，表示一个个指令最终被应用到状态机，更改其内部状态。

### Implementation: commit channel

我们的实现中，当一个 CM 被构建出来之后，它就拥有一个 commit channel，commit channel 用来向调用方（caller）返回已经被 commit 的指令：commitChan chan <- CommitEntry。

CommitEntry 的定义如下：

```java
// CommitEntry is the data reported by Raft to the commit channel. Each commit
// entry notifies the client that consensus was reached on a command and it can
// be applied to the client's state machine.
type CommitEntry struct {
  // Command is the client command being committed.
  Command interface{}
  // Index is the log index at which the client command is committed.
  Index int
  // Term is the Raft term at which the client command is committed.
  Term int
}
```

使用 channel 实现这个功能是一种设计思路，但不是唯一的方法。我们也可以使用 callback 方法；当构建一个 CM 时调用方可以注册回调方法，当我们 commit 一个指令之后就回调那个方法通知调用方。

我们很快会看到代码是如何通过 channel 发送 entries 的；但首先，我们需要讨论下 Raft server 是如何复制指令
并如何确认指令是否应该被 commit 的。

### Raft log

Raft log 在这一系列文章中已经被提到很多次了，但是我们还没有具体讲解过它。Log 简单来说就是应该被应用到状态机里的一连串的指令流（commands）；有些场合下，状态机也要能根据 log 从初始化状态完成重建（replay）。在正常情况下，所有 Raft peers 的 log 都是相同的；当 leader 收到一个新指令后，它将指令放入自己的 log 中然后将它复制给其他 followers。Follower 收到请求后也将指令放入自己的 log 中并向 leader 返回响应，leader 处保存了已经安全复制到多数派节点的最新的 log index 值。

Raft 论文中有一些像下文一样的 log 细节图：

![](https://eli.thegreenplace.net/images/2020/logdiagram.png)

每一个格子内是一个 log entry；格子顶端是该 entry 被塞进 log 时的 term 值（就是第二篇文章叙述的那个 term）。格子底部是该 entry 包含的 kv 指令。每一个 log entry 都有一个线性递增的 index，上图中就是格子上方的数值。格子的颜色是另一种区别 term 的方式。

如果这个 log 序列被应用到一个空的 kv 数据库后，最后的结果是这个 kv 数据库中 x = 4，y = 7。

在我们的实现中，log entry 的定义如下：

```java
type LogEntry struct {
  Command interface{}
  Term    int
}
```

每个 CM 的 log 属性很简单，就是 log []LogEntry 数组。客户端通常并不关心 terms；但是 terms 对于 Raft 的正确性来说确实很重要，因此在阅读代码时必须时刻牢记 term 的概念。

### Submitting new commands

让我们从新面孔—— Submit 方法开始讲解，它的作用是让客户端可以发送（submit）新指令：

```java
func (cm *ConsensusModule) Submit(command interface{}) bool {
  cm.mu.Lock()
  defer cm.mu.Unlock()
  cm.dlog("Submit received by %v: %v", cm.state, command)
  if cm.state == Leader {
    cm.log = append(cm.log, LogEntry{Command: command, Term: cm.currentTerm})
    cm.dlog("... log=%v", cm.log)
    return true
  }
  return false
}
```

很简单明了，如果 CM 是个 leader，新指令就被塞进 log 并且返回 true。否则，指令被忽略并返回 false。

Q：如果 Submit 方法返回了 true，对客户端来说是否就保证了一个新指令已经 submit 给 leader 了呢？

A：很遗憾不能保证。有一些很罕见的场景下，leader 可能会跟其他 Raft server 发生网络分区，其他节点会选举出新的 leader。但客户端由于分区的原因可能一直连接老的 leader。客户端要等待一段时间直到自己 submit 的指令出现在 commit channel 中，如果没出现，说明它可能连接到了错误的 leader 上，客户端需要重新寻找不同的节点发起重试。

### Replicating log entries

我们刚刚看到了，一个新的指令会被 append 到 log 的尾部。那这个指令如何到达 follower 呢？Leader 后续的动作在 Raft 论文中清晰地描述过，具体是在 Figure2 “Rules for Servers” 章节。我们的实现位于 leaderSendHeartbeat 方法，下面是改动后的新代码：

```java
func (cm *ConsensusModule) leaderSendHeartbeats() {
  cm.mu.Lock()
  savedCurrentTerm := cm.currentTerm
  cm.mu.Unlock()
  for _, peerId := range cm.peerIds {
    go func(peerId int) {
      cm.mu.Lock()
      ni := cm.nextIndex[peerId]
      prevLogIndex := ni - 1
      prevLogTerm := -1
      if prevLogIndex >= 0 {
        prevLogTerm = cm.log[prevLogIndex].Term
      }
      entries := cm.log[ni:]
      args := AppendEntriesArgs{
        Term:         savedCurrentTerm,
        LeaderId:     cm.id,
        PrevLogIndex: prevLogIndex,
        PrevLogTerm:  prevLogTerm,
        Entries:      entries,
        LeaderCommit: cm.commitIndex,
      }
      cm.mu.Unlock()
      cm.dlog("sending AppendEntries to %v: ni=%d, args=%+v", peerId, ni, args)
      var reply AppendEntriesReply
      if err := cm.server.Call(peerId, "ConsensusModule.AppendEntries", args, &reply); err == nil {
        cm.mu.Lock()
        defer cm.mu.Unlock()
        if reply.Term > savedCurrentTerm {
          cm.dlog("term out of date in heartbeat reply")
          cm.becomeFollower(reply.Term)
          return
        }
        if cm.state == Leader && savedCurrentTerm == reply.Term {
          if reply.Success {
            cm.nextIndex[peerId] = ni + len(entries)
            cm.matchIndex[peerId] = cm.nextIndex[peerId] - 1
            cm.dlog("AppendEntries reply from %d success: nextIndex := %v, matchIndex := %v", peerId, cm.nextIndex, cm.matchIndex)
            savedCommitIndex := cm.commitIndex
            for i := cm.commitIndex + 1; i < len(cm.log); i++ {
              if cm.log[i].Term == cm.currentTerm {
                matchCount := 1
                for _, peerId := range cm.peerIds {
                  if cm.matchIndex[peerId] >= i {
                    matchCount++
                  }
                }
                if matchCount*2 > len(cm.peerIds)+1 {
                  cm.commitIndex = i
                }
              }
            }
            if cm.commitIndex != savedCommitIndex {
              cm.dlog("leader sets commitIndex := %d", cm.commitIndex)
              cm.newCommitReadyChan <- struct{}{}
            }
          } else {
            cm.nextIndex[peerId] = ni - 1
            cm.dlog("AppendEntries reply from %d !success: nextIndex := %d", peerId, ni-1)
          }
        }
      }
    }(peerId)
  }
}
```

这段代码比我们在第二篇文章中写的要复杂很多，但它是完全按照论文描述实现的：

- AE RPC 的属性现在更丰富了：可以阅读论文 Figure2 查阅它们的定义。
- AE 的响应结果中新增了 success 属性，来告诉 leader 目标 follower 是否能匹配上 prevLogIndex 和 prevLogTerm。基于这个响应，leader 会实时更新自己内部维护的目标 follower 的 nextIndex 信息。
- commitIndex 会根据成功复制指令的 follower 数目来更新，如果一个 log entry 被成功复制到多数派节点，则将 commitIndex 更新为该 entry 的 index。

这段代码非常重要，因为它跟我们上面谈到的客户端交互逻辑紧密相关：

```java
if cm.commitIndex != savedCommitIndex {
  cm.dlog("leader sets commitIndex := %d", cm.commitIndex)
  cm.newCommitReadyChan <- struct{}{}
}
```

newCommitReadyChan 是一个被 CM 内部用来发送信号的 channel，信号的含义就是已经有新的、准备好发往 commit channel 告诉客户端的新 log entries 了。下面这个方法允许在一个 goroutine 中，随着 CM 启动而启动：

```java
func (cm *ConsensusModule) commitChanSender() {
  for range cm.newCommitReadyChan {
    // Find which entries we have to apply.
    cm.mu.Lock()
    savedTerm := cm.currentTerm
    savedLastApplied := cm.lastApplied
    var entries []LogEntry
    if cm.commitIndex > cm.lastApplied {
      entries = cm.log[cm.lastApplied+1 : cm.commitIndex+1]
      cm.lastApplied = cm.commitIndex
    }
    cm.mu.Unlock()
    cm.dlog("commitChanSender entries=%v, savedLastApplied=%d", entries, savedLastApplied)
    for i, entry := range entries {
      cm.commitChan <- CommitEntry{
        Command: entry.Command,
        Index:   savedLastApplied + i + 1,
        Term:    savedTerm,
      }
    }
  }
  cm.dlog("commitChanSender done")
}
```

这个方法内部更新 lastApplied 这个变量来标记目前哪些 index 已经发往了客户端，后续就只会发送更新的。

### Updating logs in followers

我们已经看到了 leader 如何处理新的 log entry。现在是时候看看 follower 的代码了。特别是，AE RPC 请求。在下面的代码 sample 中，高亮的部分是跟第一篇文章中实现相比，不同的地方：

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
    // 高亮 start
    
    // Does our log contain an entry at PrevLogIndex whose term matches
    // PrevLogTerm? Note that in the extreme case of PrevLogIndex=-1 this is
    // vacuously true.
    if args.PrevLogIndex == -1 ||
      (args.PrevLogIndex < len(cm.log) && args.PrevLogTerm == cm.log[args.PrevLogIndex].Term) {
      reply.Success = true
      // Find an insertion point - where there's a term mismatch between
      // the existing log starting at PrevLogIndex+1 and the new entries sent
      // in the RPC.
      logInsertIndex := args.PrevLogIndex + 1
      newEntriesIndex := 0
      for {
        if logInsertIndex >= len(cm.log) || newEntriesIndex >= len(args.Entries) {
          break
        }
        if cm.log[logInsertIndex].Term != args.Entries[newEntriesIndex].Term {
          break
        }
        logInsertIndex++
        newEntriesIndex++
      }
      // At the end of this loop:
      // - logInsertIndex points at the end of the log, or an index where the
      //   term mismatches with an entry from the leader
      // - newEntriesIndex points at the end of Entries, or an index where the
      //   term mismatches with the corresponding log entry
      if newEntriesIndex < len(args.Entries) {
        cm.dlog("... inserting entries %v from index %d", args.Entries[newEntriesIndex:], logInsertIndex)
        cm.log = append(cm.log[:logInsertIndex], args.Entries[newEntriesIndex:]...)
        cm.dlog("... log is now: %v", cm.log)
      }
      // Set commit index.
      if args.LeaderCommit > cm.commitIndex {
        cm.commitIndex = intMin(args.LeaderCommit, len(cm.log)-1)
        cm.dlog("... setting commitIndex=%d", cm.commitIndex)
        cm.newCommitReadyChan <- struct{}{}
      }
      
      // 高亮 end
    
    }
  }
  reply.Term = cm.currentTerm
  cm.dlog("AppendEntries reply: %+v", *reply)
  return nil
}
```

这段代码也是完全按照论文 Figure2 实现的，并附带了十分详细的注释。

当 follower 意识到 leader 的 LeaderCommit 比自己的 cm.commitIndex 大时，会向 newCommitReadyChannel 发送信号；这就是 follower 确认 leader 已经 commit 了这些 log entries 的时刻。

当一个 leader 用 AE 请求发送 log entries 时，会发生下面这些事情：

- 一个 follower 将新的 entries 塞进自己的 log 中，并向 leader 返回 success。
- leader 收到成功的响应之后，更新自己保存的该 follower 的 matchIndex。当收到了多数派 follower 都针对某个 matchIndex 返回成功的话，leader 就更新自己的 commitIndex 并将之附带在下一次 AE 请求中发送给 follower。
- 当 follower 从 leader 处收到 AE 发现 leaderCommit 比他们之前接收到的值更新的话，他们知道 leader 已经 commit 了一些新的 log entries，因此他们可以将这些 log entries 应用给自己的状态机（代码中即向 newCommitReadyChannel 发送信号）。

Q：commit 一个新指令需要多少论 RPC 调用？

A：两轮，第一轮需要 leader 将新的指令 log entries 发送给 follower，follower 将之塞到自己的 log 中并向 leader 返回成功。当 leader 处理 AE 请求时，它可能会根据收集到的 follower 的响应结果更新自己的 commit index。第二轮中 leader 会将更新之后的 commit index 通过 AE 发送给 followers，让他们意识到有新的 log entries 可以被提交了。你可以回过头阅读代码中关于这些 RPC 请求的部分。

### Election safety

目前为止，我们已经阅读完实现 log replication 的新增代码了。但是不仅限于指令复制，logs 还会对 leader 选举产生影响。Raft 论文中在 5.4.1 节描述过，称这种机制为 Election restriction。Raft 通过这种机制保证集群中拥有最新 log 的 peer 才能当选为 leader，什么叫拥有 “最新的 log” 呢，并不单纯意味最大的 log index，而是已经在集群多数派复制完成的最大的 log index。

为了实现该机制，RVs 请求会包含 lastLogIndex 和 lastLogTerm 两个属性。当一个 candidate 发送 RV 请求时，它会在消息体中附带这两个信息，两个值的来源是自己拥有的最新的 log entry。Followers 会从 RV 里获取这两个值，并跟自己本地的 log entry 进行比较，来判断该 candidate 的日志是否足够新，是否能被选举为 leader。

下面高亮的是 startElection 方法中新增的代码：

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
      
      // 高亮 start
      
      cm.mu.Lock()
      savedLastLogIndex, savedLastLogTerm := cm.lastLogIndexAndTerm()
      cm.mu.Unlock()
      args := RequestVoteArgs{
        Term:         savedCurrentTerm,
        CandidateId:  cm.id,
        LastLogIndex: savedLastLogIndex,
        LastLogTerm:  savedLastLogTerm,
      }
      cm.dlog("sending RequestVote to %d: %+v", peerId, args)
      var reply RequestVoteReply
      
      // 高亮 end
      
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

其中，lastLogIndexAndTerm 是一个新的辅助方法：

```java
// lastLogIndexAndTerm returns the last log index and the last log entry's term
// (or -1 if there's no log) for this server.
// Expects cm.mu to be locked.
func (cm *ConsensusModule) lastLogIndexAndTerm() (int, int) {
  if len(cm.log) > 0 {
    lastIndex := len(cm.log) - 1
    return lastIndex, cm.log[lastIndex].Term
  } else {
    return -1, -1
  }
}
```

需要提醒的是，我们的实现中 log 的 index 是从 0 开始的，论文中是从 1 开始的。因此这里用 -1 标识没有任何日志的情况。

下面是一个代码更新过的 RV handler 实现，包含了对于 candidate 选举资格的判断，其中新增的代码同样高亮显示：

```java
func (cm *ConsensusModule) RequestVote(args RequestVoteArgs, reply *RequestVoteReply) error {
  cm.mu.Lock()
  defer cm.mu.Unlock()
  if cm.state == Dead {
    return nil
  }
  
  // 高亮 start
  
  lastLogIndex, lastLogTerm := cm.lastLogIndexAndTerm()
  cm.dlog("RequestVote: %+v [currentTerm=%d, votedFor=%d, log index/term=(%d, %d)]", args, cm.currentTerm, cm.votedFor, lastLogIndex, lastLogTerm)
  // 高亮 end
  
  if args.Term > cm.currentTerm {
    cm.dlog("... term out of date in RequestVote")
    cm.becomeFollower(args.Term)
  }
  if cm.currentTerm == args.Term &&
    
    // 高亮 start
    
    (cm.votedFor == -1 || cm.votedFor == args.CandidateId) &&
    (args.LastLogTerm > lastLogTerm ||
      (args.LastLogTerm == lastLogTerm && args.LastLogIndex >= lastLogIndex)) {
    
    // 高亮 end
    
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

### Revisiting the "runaway server" scenario

在第二篇文章中，我们描述过一个场景，server B 处于一个三机集群中，并由于网络分区致使它跟集群其他节点断连了一段时间，导致它变为 candidate 并不停发起选举。当它重新连回集群后，由于它的 term 非常大，可能导致它成功当选 leader。

现在我们回顾下这个场景，思考下由于增加了 log replication，现在会发生什么？

即使 B 重连回集群后 term 确实是最大的，它也确实会自己发起选举，leader A 收到的 AE 心跳响应中，也确实会察觉 B 的 term 比自己的还要大，但是 B 不可能当选为 leader 因为它的日志比 A 和 C 的都要少。这就是上述我们讲解的 election restriction 的作用。Leader 仅会在 A 或者 C 之间产生，因此集群产生的混乱被限制到最小。

如果你还是觉得疑惑，为什么 B 重连回集群之后，整个集群一定要发起新的选举呢？Ongaro 的博士论文中讨论了这个问题，位于标题为 “Preventing disruptions when a server rejoins a cluster” 的章节中。通常解决这种问题的举措是 “pre-vote”，即当 server 转变为 candidate 时做一些前置检查。

由于这是并不常见的场景，我不会花费太多时间讨论，具体实现 Raft 时你可以将之作为一个优化点。更多详细的内容你可以阅读他的学位论文，链接放在 Raft 官网上。

### Some Q&A

本节会回答一些在我们学习和实现 Raft 的过程中可能产生的典型问题。如果你还有其他问题，可以给我发邮件——我会收集频率高的问题，之后再更新文档。

Q：为什么 commitIndex 和 lastApplied 要分成两个参数，我们不能就使用 commitIndex 作为 RPC 的参数么？然后仅向客户端返回这些已提交的指令。

A：分成两个参数可以让需要被快速处理的请求（例如 RPC handling）和可以缓慢处理的请求（例如向客户端返回指令）解耦。思考下当一个 follower 收到 AE 并发现 leader 的 commitIndex 比自己的更大时会发生什么？此时它会发送一连串指令到 commit channel。但是往一个 channel 发送消息可能是个阻塞操作，但同时我们又希望尽快响应 RPC 请求。lastApplied 参数就帮助我们完成解耦。RPC 仅更新 commitIndex，运行 commitChanSender 方法的 goroutine 在后台观察到变化后就在空闲时将新的 committed commands 应用到状态机中。

你可能想知道这是否同样可以类比到 newCommitReadyChan channel，这是个非常好的想法。这个 channel 是有缓冲区的，但是由于我们控制了 channel 的两端，因此我们可以设置一个小缓冲区来保证高性能的同时没有阻塞。但在一些极端场景下，让应用到状态机的过程非常缓慢时，可能会造成 RPCs 处理延时，因此 Raft 不可能保持一个无限长的缓冲区。这并不是完全的坏事，因为它会创建一个自然的背压机制。

Q：为什么我们在 leader 处需要为每一个 peer 维护 nextIndex 和 matchIndex？

A：如果仅有一个 matchIndex 算法也可以正常工作，但在一些场景下会比较低效。考虑个 leadership 变更的场景，新的 leader 无法从他的 peers 得到任何信息并初始化 matchIndex 为 -1。然后它将全量的日志发送给每个 follower。但是很可能 follower 已经有了绝大部分的日志，并不需要 leader 全量发送一遍；nextIndex 可以帮助 leader 更细粒度地重建 follower 的日志，而不是直接发送全量。

### What's Next

再强调一遍，我强烈建议你自己试玩一下代码——跑一跑单测自己观察下日志输出。

此时此刻，我们已经几乎实现了 Raft 算法，除了我们还未实现持久化以外。这表明我们的算法还无法容忍 crash，不论是 server crash 还是重启等。

Raft 算法也有关于这部分的内容，我们将在第四篇文章中提到。增加持久化后，我们能够进行更隐蔽的测试，包括服务器在最坏的时间崩溃。

另外，第四篇文章还将提及一些实现上的优化点，最关键的是，当 leader 有更新的信息时如何更迅速地通知 followers；现在的实现中 leader 仅是每隔 50ms 发送 AEs。这会在后续的实现中被优化。