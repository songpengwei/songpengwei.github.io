---
title: 6.824 - raft 实现（二）：日志同步（log replication）
updated: 2018-08-29 20:06
tag: raft, 6.824 
---

## 前言
上一次在做完 lab2a 即 raft 的 leader 选举之后，一直卡在日志同步这一块（log replication）；直到昨晚进行了一下 appendEntries 的优化（prevLog 不匹配时，一下跳过本 term 所有 logEntries），一直困扰的 TestBackup2B 竟然神奇 Passed 的了。跑了两遍还不大信，特地将其改回去，看到果然 Fail 才放心下来，看来是效率太低超时了。

趁着还新鲜，索性今晚就将这一段时间的血泪史记下来吧。

## 实验概述
6.824是MIT的一门分布式课程，我跟的是[2018 spring](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html) 。在第二个实验中要求简单实现一个分布式一致性协议--[raft](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)。

这是一个专为方便教学和工程实现所设计的协议，它将协议拆解为几个相对独立的模块--leader选举，日志同步，安全保证。论文里图二基本给出了Raft的所有实现细节，可谓字字珠玑。但也因为太微言大义了，导致有些状态转换分散在不同描述中，假如你真只照着这幅图实现，很容易遗漏些细节。

## 日志同步（Log Replication）
这是lab2B的内容，又是反反复复，猜坑，排雷，放弃，捡起一月余才将日志同步这一块完成，最终代码不到七百行（694），其间有些实现细节还受过网上的启发。其间状态类似于网上看到的一幅调侃程序猿的漫画——啊，出bug了，为什么啊，改改改；还是不对，改改改；啊终于对了，可是，又是为什么啊？

最终 Passed 版本我觉得还待改进，有些地方隐隐然感觉略多余。但暂时先高兴会，把心得赶到纸上来，毕竟老年人脑袋的内存实在有限，而且易挥发。

上一篇 leader 选举中主要是对实现技巧做了些记录，这一篇主要是对实现各个逻辑细节进行探讨。

### 大致流程

当初论文大概看明白了，觉得各个细节都搞懂了，但是整个流程在脑中还是转不起来。现在想来，Leader上线后，日志同步过程大概是这样：

1. Leader 选出后，进行初始化，主要是对 nextIndex 数组和 matchIndex 数组的初始化，可能有多种思路，但是我是这么做的：所有 `nextIndex = len(rf.log)` ，所有 `matchIndex = 0`
2. 然后立即开始心跳，即 AppendEntries，论文上说开始发一个不带内容的心跳就行，但是我的实现中还是发了带一个 logEntry 的心跳。这取决于我上面的初始化方法以及下面的参数构造策略。
3. 同步过程，我将其分为对 match 位置的 **试探阶段** 和 match 后的**传送阶段**；即首先通过每次前移 `prevLogIndex+prevLogTerm` 匹配上 Follower 中的某个 `logEntry`，然后 Leader 将匹配到的 `logEntry` 之后的 entries 一次性送给 Follower。
4. 每次参数构造，其他几个都比较确定，主要考虑三个字段 `prevLogIndex`，`prevLogTerm` 和 `entires`。其中 `prevLogTerm` 又由 `prevLogIndex` 决定，因此主要考虑 `prevLogIndex` 和 `entries`。对于 `prevLogIndex`，在 **试探阶段** 我将其定为 `nextIndex-1`，这么做是为了在试探时降低不必要的 `entries`传输；在**传送阶段**，可以将 `prevLogIndex` 定为 `matchIndex`，也即已经匹配上的最后一条数据。对于 `entries`，即取 Leader log 中  `[prevIndex+1, min(nextIndex, len(rf.log)-1)]` 的一个闭区间。
5. 这样，每次试探的时候是不传送数据的（即为 [] ，因为此时`prevLogIndex` = `nextIndex`-1），在传送阶段可以一次性的将所有匹配之后的 Leader 上的 logEntries 全部传送过去。如果没有新日志了，几个变量将维持在：`nextIndex = len(rf.log)；prevLogIndex = matchIndex = len(rf.log)-1`，`entries` 为 []；

其他点就是附着于此流程之上的一些细节：

1.  定时检测大多数 logEntry match Index，以决定是否要进行 commit，即前移 Leader 的 `commitIndex` ，并且在以后的 `AppendEntries` 的 RPC 中将其同步给各个 Follower。
2. 同时另一个线程要检测 `lastApplied` 是否跟上了 `commitIndex` ，保证将已提交 log 及时应用到状态机中。这两个变量分开我觉得主要是为了逻辑上的解耦——在 Leader 上是 Leader 主动检测来更新 `commitIndex` 的，在 Follower 上，是被动接受 Leader 消息来更新的。
3. Leader 接受了新的 cmd 后，需要将 `matchIndex` 数组中本身对应的位置更新，因为它本身也是最后计算大多数时的一票。
4. Follower 连不上之后，Leader要及时终止执行回调后边内容。



### AppendEntries 心跳&同步

主要对应上面说的两个阶段，试探阶段和传送阶段；其他要注意的就是上一篇提到的加锁和状态自检。

```go
// need handle the reply
if reply.Term > rf.currentTerm {
    rf.becomeFollower(reply.Term)
} else {
    if !reply.Success {
        // roll back per term every time
        nextIndex := rf.nextIndex[server] - 1
        for nextIndex > 1 && rf.log[nextIndex].Term == args.PrevLogTerm  {
            nextIndex--
        }
        rf.nextIndex[server] = nextIndex

    } else {
        // if match, sync all after
        rf.matchIndex[server] = args.PrevLogIndex + len(args.Entries)
        rf.nextIndex[server] = len(rf.log)
    }
}
```

然后就是参数构造，对于 `prevIndex ` 来说，也是两个阶段不同构造。然后取合适窗口的 `entries`。

```go
func (rf *Raft) constructAppendEntriesArg(idx int) *AppendEntriesArgs {
	prevLogIndex := 0
	if rf.matchIndex[idx] == 0 {
		prevLogIndex = rf.nextIndex[idx] - 1	// try to find a match
	} else {
		prevLogIndex = rf.matchIndex[idx]		// after match to sync
	}
	prevLogTerm := rf.log[prevLogIndex].Term

	// the need to replica window [prevLogIndex+1, nextIndex)
	var entries []*LogEntry
	start := prevLogIndex + 1
	end := min(rf.nextIndex[idx], len(rf.log)-1)
	for i := start; i <= end; i++ {
		entries = append(entries, rf.log[i])
	}

	return &AppendEntriesArgs{rf.currentTerm, rf.me, prevLogIndex,
		prevLogTerm, entries, rf.commitIndex}
}
```

最后 AppendLogEntries 回调，看到先前 term 的请求，直接拒绝，这没什么好说的，注意将自己的 term 带回去就行。此外，无论 args.Term > or = rf.currentTerm ，收到 AppendEntries 的 peer 都要变成 Follower，并且重置 timer。此外在匹配 logEntry 的时候，要注意进行截断。

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	// reject the append entries
	if args.Term < rf.currentTerm {
		reply.Success = false
		return
	}

	rf.leaderId = args.LeaderId
	rf.becomeFollower(args.Term)

	if args.PrevLogIndex >= len(rf.log) {
		reply.Success = false
	} else if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
		reply.Success = false
		rf.log = rf.log[:args.PrevLogIndex]
	} else {
		reply.Success = true
        
		// delete not match and append new ones
		if len(args.Entries) > 0 {
			rf.log = append(rf.log[:args.PrevLogIndex+1], args.Entries...)
		}

		// commit index
		if args.LeaderCommit > rf.commitIndex {
			rf.commitIndex = min(args.LeaderCommit, len(rf.log)-1)
		}
	}
}
```



### CommitIndex 更新

主要依赖 leader 的 matchIndex[] ，具体做法是将其升序排序，然后取中位数位置的 Index（当时拍脑袋想出来的，感觉很神奇）。另一个要注意的点是论文中着重强调的，就是只能提交本 Term 中的 logEntry，这是为了 Leader 你方唱罢我方登场引起的反复覆盖。

```go
func (rf *Raft) checkCommitIndex() {
	peersCount := len(rf.peers)
	matchIndexList := make([]int, peersCount)
	copy(matchIndexList, rf.matchIndex)
	sort.Ints(matchIndexList)

	// match index before the "majority" are all matched by majority peers
	// before we inc commitIndex, we must check if its term match currentTerm
	majority := peersCount / 2
    peerMatchIndex := matchIndexList[majority]
    if peerMatchIndex > rf.commitIndex && rf.log[peerMatchIndex].Term == rf.currentTerm {
        rf.commitIndex = peerMatchIndex
    }
}
```



### 响应投票

每个 peer 在每个 term 最多投一票，但是每次更新 term 后，可将 votedFor 赋值为 -1，即又可以投票了。这个 case 发生在响应 candidate 要求投票时，如果他已经投了票，这是好像不能投票，但是发现 term 没有 candidate 的大，那么需要立即变为 Follower ，并且给该 candidate 投票。

``` go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
    
	reply.Term = rf.currentTerm

	// once find a peer with higher term, follow
	if args.Term > rf.currentTerm {
		rf.becomeFollower(args.Term)
	}

	// compare term and test if it voted
	if args.Term < rf.currentTerm || rf.votedFor != -1 {
		reply.VotedGranted = false
		return
	}

	// compare the last log entry
	lastIndex := len(rf.log) - 1
	lastLogTerm := rf.log[lastIndex].Term
	if args.LastLogTerm > lastLogTerm ||
		args.LastLogTerm == lastLogTerm && args.LastLogIndex >= lastIndex {
		reply.VotedGranted = true

		// convert to follower
		rf.becomeFollower(args.Term)
		rf.votedFor = args.CandidateId // do not forget
	} else {
		reply.VotedGranted = false
	}
}
```

这里涉及到 becomeFollower 的实现，即更改状态，然后重设 timer，并且根据 term 是否更新来决定是否可以投票。

``` go
func (rf *Raft) becomeFollower(term int) {
	DPrintf("%d[%d] become follower", rf.me, term)
	rf.resetElectionTimer()
	rf.state = Follower
	if term > rf.currentTerm {
		rf.votedFor = -1
	}
	rf.currentTerm = term
}
```

