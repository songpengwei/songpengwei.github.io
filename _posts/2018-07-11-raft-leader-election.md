---
title: 6.824 - raft 实现（一）：leader选举
updated: 2018-07-11 22:13
tag: raft, 6.824 
---

## 概述
记录下在实现6.824 lab2 raft 的一些想法和经验，聊以备忘。

## 实验概述
6.824是MIT的一门分布式课程，我跟的是[2018 spring](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html) 。在第二个实验中要求简单实现一个分布式一致性协议--[raft](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)。

这是一个专为方便教学和工程实现所设计的协议，它将协议拆解为几个相对独立的模块--leader选举，log复制，安全保证。论文里图二基本给出了Raft的所有实现细节，可谓字字珠玑。但也因为太微言大义了，导致有些状态转换分散在不同描述中，假如你真只照着这幅图实现，很容易遗漏些细节。

## leader选举
这是lab2A的内容，说来惭愧，从开始构思到测试用例pass，前前后后拖了两个多月。虽然只是晚上和周末写写，但是也的确进展缓慢，不过收获颇多。暗想要是本科实验也这么来，可能就真如知乎所说，一学期顶多学两门课，回想大三时候四门四学分的课，从课程设计上来说就是注定要我们划水的。

吐槽完毕，说说踩过的坑。
### 定时器
raft主要有两个event loop，一个是（Follower，Candidate）超时发起选举，一个是（Leader）定期心跳（有时捎带日志同步）。最容易想到的就是本科写大作业无脑用的loop+sleep，即外层一个while true，内层用一个稍小（但比electionTimeout和heartbeatInterval至少小一个数量级才够用）的时间间隔（比如说t）sleep，来周期性检测时间节点（needElection，heartbeat）的到来。

但是我老强迫症的觉得，这么着不准确，误差至少是那个检测时间间隔t。这要是好多线程搞来搞去，面对这么复杂的状态变化，会不会由误差而导致错误，但是用go的timer好像很复杂的样子，每每想到这就想不清了，这么着也耽搁了一阵子。

直到后来，在另一个也在做raft的孩子提醒下才注意到，其实课程很贴心的给出了[建议](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)--还就是利用loop+sleep来周期性检测超时。这时候我茅塞顿开，悟出了上面括弧中我的注释--只要保证检测间隔小于超时间隔一两个数量级，基本上就没啥问题。这种实现的优点是简单，粗暴，直接，可控。

我在实现的时候又突然觉得好玩，弄成了两种稍有不同的实现，下附代码，为了看着清楚，都做了些简略化处理。
一种是只有一个loop：

```go
go func() {
	for {
		now := time.Now()
		if now.Sub(last) < electionTimeout {
			time.Sleep(checkGapInMs)
		} else {
			// 起初我还纠结此两句顺序，后来想通了
			// 只要startElection没有啥IO等阻塞型操作，顺序就无所谓
			startElection()
			last = time.Now()
		}
	}
}()
```

另一种是两个嵌套loop，内层loop专门用来等待（或许是联想到了CPU的忙等待）。

```go
for {
	// ping all the peers
	for s := 0; s < len(rf.peers); s++ {
		 // append entry rpc
	}(s, args)
	
	// wait until time comes
	for {
		now := time.Now()
		if now.Sub(lastPingTime) < pingGapInMs {
			continue
		} else {
			lastPingTime = time.Now()
			break
		}
	}
}
```

虽然前者更简洁，但是后者逻辑更明白；有时候简洁的代码是复用了不同的逻辑，导致语义上可能稍微有点不清楚。考虑到代码的第一要义是给人看的（啥？你说是给机器看的？咳咳，我觉得这是代码之所以为代码的前提，不然编译器不给你过啊），我觉得还是第后者好一些。

### 加锁
课程也很贴心的给出了[提示](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)，但作为一个从大一电梯大作业开始就开始‘玩锁’的人，我直接将其略过，不管三七二十一，昂首挺胸，赤膊上阵，狂撸代码。然而，记忆是会骗人的，教训是很惨痛的，竟然一直死锁而我却不知道是哪的问题。后来只得乖乖将提示看了好几遍，发现写的真是好啊。。。

其要点总结起来，就是将有全局变量读写的地方先全部给加上锁，然后再将有阻塞（rpc call等等）的地方去掉锁。

它这原则先按下不表，来说说我跳坑的两个地方：
#### 函数调用外持有了锁，在函数调用里面又重新申请：
犯这个错误的原因是咱看着那么长的代码，凭着经验，总得给他包一下吧，包的时候发现我擦全是全局变量，赶紧申请锁啊。但是后来在调用该函数的时候，忘了放在临界区了；也就是说吃着碗里的，又想盛锅里的。死锁成就1 get，废话不多讲，上代码：

```go
a.
func (rf *Raft) startElection() {
	rf.mu.Lock()
	// balalala
}

b.
rf.mu.Lock()
// balala
rf.startElection()
```

####  分支break/return忘记释放锁
`break，continue，return`这种偷偷摸摸的提前结束分支的行为，虽然我们平时干的很开心，但是它不符合人的既定对称认知啊，就导致有时候忘了处理它，什么叫不对称呢？上代码：


```go
if !check(arg) {
	return
}

for condition {
	if need {
		break
	}
	
	// because here are so many balalala
	// then break can be used to reduce indent
}
```
v.s.

```Go
if !check(arg) {
	return
} else {
	for condition {
		if !need {
			// because here are so many balalala
			// then break can be used to reduce indent
		}
	}
}
```
如果是后者，对着`if else`扫一眼对齐，函数有几个出口，一目了然。然而对于前者来说，如果分支语句淹没在巨量代码中，就很容易忘记某个猥琐角落还藏着一个出口，自然就不会上锁。

具体到我的情况，就是在candidate要票的时候，如果得到多数票，就直接变为leader；如果后面仍有人给票，我会判断当前身份是否已经是leader，如果是就直接返回，不在意这些票了，的确好邪恶。。于是，你懂得，返回前忘还回锁了。代码如下，有删节。

```go
go func(server int, args RequestVoteArgs) {
	// use args to request vote
	reply := RequestVoteReply{}
	ok := rf.sendRequestVote(server, &args, &reply)

	isBecomeLeader := false
	rf.mu.Lock()
	if rf.state != Candidate || rf.currentTerm != args.Term{
		rf.mu.Unlock() //<---就是这个邪恶的地方
		return
	}

	// check the votes
	if reply.VotedGranted {
		votes++
		DPrintf("%d get vote from %d, now votes are %d, total members are:%d",
			rf.me, server, votes, peersCount)
		if votes > peersCount/2 {
			if rf.state == Candidate {
				isBecomeLeader = true
				DPrintf("%d become leader", rf.me)
			}
			rf.state = Leader
		}
	 } else if reply.Term > rf.currentTerm {
		rf.currentTerm = reply.Term
		rf.state = Follower
		rf.votedFor = -1
	 }
	rf.mu.Unlock()

	if isBecomeLeader {
		rf.startHeartbeat()
	}
}(s, args)
```

#### Gap之后，注意自检
由于状态是在多个loop（好吧，也就两个，但是为啥觉得很多呢）中来回改变的，因此在异步/阻塞调用前后，可能身份角色（自身）和选举周期（外界）早已天翻地覆，因此需要进行自检。
说来惭愧，这个也是在多方查看资料后才意识到的。
1. 异步调用，即goroutine内外。
2. 阻塞调用，即rpc前后。

Leader进行心跳检测时候，由于是异步调用，所以需要先检测自己还是不是leader，以及是否还在自己任期（由args保存了当时的任期）；当rpc返回后，再一次进行该检查，无误，才能按照自己是leader来进行下一步动作。

```go
go func(server int, args *AppendEntriesArgs) {
	if rf.currentTerm != args.Term || rf.state != Leader {
		return
	}

	reply := &AppendEntriesReply{}
	rf.sendAppendEntries(server, args, reply)

	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.currentTerm != args.Term || rf.state != Leader {
		return
	}

	// need handle the reply
	if reply.Term > rf.currentTerm {
		rf.currentTerm = reply.Term
		rf.state = Follower
		rf.votedFor = -1
	}

```


