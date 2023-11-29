---
keywords:
  - "分布式"
title: "MIT6.5840"
date: 2023-11-14T15:48:21+08:00
lastmod: 2023-11-14T15:48:21+08:00
description: ""
draft: true
author: 椰虂serein
hideToc: false
enableToc: true
enableTocContent: false
tocFolding: false
tocLevels: ["h2", "h3", "h4"]
tags:
  - "分布式"
categories: "分布式"
img: ""
---

> 注：本博客记录本人实现MIT6.5840的思路，若有错误，请多包含

## 传送门

Raft论文：https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf

Raft实现参考实例：https://github.com/hashicorp/raft

## 基本思路概述

MIT6.5840的Lab2主要实现分布式一致性算法（Raft）。在实验任务中，主要分为领导人选举、日志复制、持久化、日志压缩四个部分。本人在完成实验的过程中，主要是从Leader、Candidate、Follower三个角色的任务出发，不断完善过程。

### 基础数据结构
数据结构主要是根据论文设计的，在某些方面与原文会有所不同，这里先不过多解释，直接上源码。
```go
type Raft struct {
	mu        sync.Mutex         
	peers     []*labrpc.ClientEnd 
	persister *Persister          
	me        int                 
	dead      int32               
	
	state       State // 节点角色
	currentTerm int // 节点当前任期
	votedFor    int // 投票对象
	log         *Log // 日志文件

	commitIndex  int //提交的最后一个日志的索引
	lastApplied  int // 发送的最后一个日志的索引
	applyCond    *sync.Cond
	applyManager *ApplyManager

	heartbeatTimeout  time.Duration
	electionTimeout   time.Duration
	lastElectionTime  time.Time
	lastHeartbeatTime time.Time
	trackers          []Tracker  //用来记录节点的日志提交相关信息

	snapshot                 []byte
	snapshotLastIncludeIndex int // 快照包含的最后一个日志的索引
	snapshotLastIncludeTerm  int // 快照包含的最后一个日志的任期
}
```
### 基本功能
![Raft basic](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE4.png)
接下来，按照论文所给的思路设计总体流程。在Raft算法中，Follower与Candidate的任务，主要是在任期结束之后发起新的选举。在选举过程中，Candidate会不断向其余节点索取投票，得到大多数票数（大于总数的1/2）的节点会成为Leader。Leader与Candidate会在选举时判断自身与交互节点的任期状态，从而采取不同的行动。Raft算法的执行通过**ticker**函数实现

```go
func (rf *Raft) ticker() {
	for !rf.killed() {
		rf.mu.Lock()
		state := rf.state
		rf.mu.Unlock()
		switch state {
		case Follower:
			fallthrough 
		case Candidate:
			if rf.pastElectionTimeout() {
				rf.StartElection()
			}
		case Leader:
			isHeartbeat := false
			if rf.pastHeartbeatTimeout() {
				isHeartbeat = true
				rf.resetHeartbeatTimer()
				rf.StartAppendEntries(isHeartbeat)
			}
			rf.StartAppendEntries(isHeartbeat)
		}
		time.Sleep(tickInterval)
	}
}
```
> Q1:这里为什么不直接switch rf.state，而要用state记录节点状态，同时这一个过程还要加锁？<br>
> A1:因为ticker函数实际上是在Make创建了Raft实体后开启的一个协程。而在此函数中，StartElection等函数中存在并发，涉及到节点状态的转换。同时考虑到节点断联的可能，若不加锁，无法保证在一段时间内（心跳到期或选举到期）节点的State永远不变。

## 任务实现

### 选举（StartElection）

![algorithm](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE2.png)

根据论文，RequestVote通信参数定义如下

```go
type RequestVoteArgs struct {
	Term         int //发送者当前任期
	CandidateID  int //候选人ID
	LastLogIndex int //最后一条日志的下标
	LastLogTerm  int //最后一条日志的任期
}
type RequestVoteReply struct {
	Term        int //接收者当前任期
	VoteGranted bool //是否收到投票
}
```
选举开始时，节点会将自己的身份转换为Candidate，先投自己一票，并且向其余所有节点同时发送拉票请求。
```go
go func(server int) {
	var reply RequestVoteReply
	ok := rf.sendRequestVote(server, &args, &reply)
	if !ok || !reply.VoteGranted { //通信失败或者没有收到选票，直接结束
		return
	}
	rf.mu.Lock()
	defer rf.mu.Unlock()
     //投票人任期过期，重置投票人的状态，投票人在数据达成一致前不参与投票
	if reply.Term < rf.currentTerm {
		return
	}
	// 统计票数
	votes++
	if done || votes <= len(rf.peers)/2 {
		// 在成为leader之前如果投票数不足需要继续收集选票
		// 在成为leader的那一刻，立刻结束选举，发送心跳
		return
	}
	done = true
	if rf.state != Candidate || rf.currentTerm != term {
		return
	}
	rf.changeState(Leader)
}(i)
```

> Q2：`if rf.state != Candidate || rf.currentTerm != term` 判断的是哪种情况？<br>
A2：首先要明确，当一个节点断联的时候，节点本身不一定是无法运行的状态，可能只是RPC通信错误导致的。此时，如果一个节点断联，则会有两种情况。若节点为Leader，则会不断发送心跳，但不会收到回复，也就不会更新任期。在重联后，会因为自身任期旧而失去Leader身份重置为Follower。若节点为Follower，在断联时接收不到心跳，故自身的选举计时器会不断超时，导致自己不停发起选举，重联后自己任期一定为最新，可以成为Leader。因为在拉票之前，节点会将自己的身份转变为Candidate，这里的条件就是判断节点是否在选举过程中断联，如果断联，重置前不会参与选举。

#### 拉票

拉票是整个选举的核心内容，决定了每个节点的角色。这个动作对于所有节点而言是平等的。整个过程主要会出现下列几种情况：

1. 发送者任期过期，拒绝投票

   ```go
   if args.Term < rf.currentTerm {
   	  reply.VoteGranted = false
   	  return
   }
   ```

2. 接收者任期过期，跟新接收者任期
   ```go
   if args.Term > rf.currentTerm {
	  rf.currentTerm = args.Term
	  reply.Term = rf.currentTerm
	  rf.votedFor = -1
	  rf.state = Follower
   }
   ```
3. 两者任期相同，此时要求发送者的日志和接收者一样新（具体表现为Candidate最后一条日志任期比接收者更大或者在任期相同的情况下日志索引更大）
   ```go
    update := false
    update = update || args.LastLogTerm > rf.getLastLogTerm()
    update = update || args.LastLogTerm == rf.getLastLogTerm() && args.LastLogIndex >= rf.log.LastLogIndex
   ```
   ```go
	if (rf.votedFor == -1 || rf.votedFor == args.CandidateId) && update {
		rf.votedFor = args.CandidateId
		rf.state = Follower
		rf.resetElectionTimer()
		rf.persist()
	} else {
		reply.VoteGranted = false
	}
	```
> 这里的日志比较其实可以类比为文档，选取文档时，版本（日志的任期）更高的文档往往比版本低的更贴近于技术的更新，而在版本相同时，我们更趋向于选取内容更全（日志的索引）的文档作为参考。
## 心跳
先回看一下ticker中Leader的动作
```go
case Leader:
	isHeartbeat := false
	// 心跳定时器过期发送心跳，否则发送日志
	if rf.pastHeartbeatTimeout() {
		isHeartbeat = true
		rf.resetHeartbeatTimer()
		rf.StartAppendEntries(isHeartbeat)
	}
	rf.StartAppendEntries(isHeartbeat)
}
```
发送心跳是*leader*的动作,即根据回复结果，若自身任期落后，将自身状态重设为Follower；若接收方任期落后或者通信成功，则不做任何处理。此处就不再列出源码。
>`isHeartbeat`在这里作为心跳的标记。不过，Raft中心跳信号的实现实际上是通过在复制日志时发送一个空的日志文件来实现的，因此这里的标记并不是绝对必要的。<br>

接下来从follower角度看看对心跳消息的处理

```go
func (rf *Raft) HandleHeartbeat(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.FollowerTerm = rf.currentTerm
	reply.Success = true
    //拒绝旧leader
	if args.LeaderTerm < rf.currentTerm {
		reply.Success = false
		reply.FollowerTerm = rf.currentTerm
		return
	}
	rf.resetElectionTimer()

	//转变身份为Follower
	rf.state = Follower
	rf.votedFor = args.LeaderID
	if args.LeaderTerm > rf.currentTerm {
		rf.votedFor = -1
		rf.currentTerm = args.LeaderTerm
		reply.FollowerTerm = rf.currentTerm
	}
}
```

## 日志处理
日志的处理包括日志的复制、提交与压缩，主要的难点集中在对于日志文件的索引及任期的判断与处理。<br>
日志由Entry构成，每个Entry又有自身的任期。

```go
type Entry struct {
	Term    int
	Command interface{}
}

type Log struct {
	Entries       []Entry // 初始状态为空，不包含快照
	FirstLogIndex int
	LastLogIndex  int
}
```
为了追踪每个节点的日志提交的相关参数，这里设计tracker进行存储
```go
type Tracker struct {
	nextIndex  int
	matchIndex int
}
```
另外，日志快照相关参数设计如下
```go
snapshot                 []byte
snapshotLastIncludeIndex int
snapshotLastIncludeTerm  int
```

### 日志复制
同样，根据论文，先设计出信息交互的数据结构
```go
type AppendEntriesArgs struct {
	LeaderTerm   int //领导人任期
	LeaderID     int //领导人ID
	PrevLogIndex int //要发送的所有日志中第一个日志的前一位索引
	PrevLogTerm  int //要发送的所有日志中第一个日志的前一位索引对应日志的任期
	Entries      []Entry //日志本体
	LeaderCommit int //leader已经提交的最大日志索引
}

type AppendEntriesReply struct {
	FollowerTerm int //跟随者任期
	Success      bool //是否能够成功复制
	PrevLogIndex int  //对于follower可以直接复制的前一位索引
	PrevLogTerm  int  //对于对于follower可以直接复制的前一位索引对应日志的任期
}
```
日志复制的过程相对复杂，此处从follower节点的角度分类讨论处理方式：
1. leader任期过旧，拒绝复制日志
   ```go
   if args.LeaderTerm < rf.currentTerm {
		reply.Success = false
		return
	}
   ```
2. 接收者自身任期过旧，重置自身状态
   ```go
   if args.LeaderTerm > rf.currentTerm {
		rf.votedFor = -1
		rf.currentTerm = args.LeaderTerm
		reply.FollowerTerm = rf.currentTerm
	}
   ```
3. 本身日志为空
   这里分两种情况，一是节点刚刚初始化完成，没有日志记录，此时面对发来的日志记录，可以全部接收；二是刚经历了一次日志压缩，日志信息存储在快照中，自身的日志为空，此时若快照存储的日志与要复制的日志刚好接上，则可以接受全部记录。<br>
   同样的，如果日志为空，但是要复制的日志与快照没有办法相接，则不会接受请求
   ```go
   if args.PrevLogIndex == rf.snapshotLastIncludeIndex {
		rf.log.appendLogs(args.Entries...)
		reply.FollowerTerm = rf.currentTerm
		reply.Success = true
		reply.PrevLogIndex = rf.log.LastLogIndex
		reply.PrevLogTerm = rf.getLastLogTerm()
		return
	} else {
		reply.FollowerTerm = rf.currentTerm
		reply.Success = false
		reply.PrevLogIndex = rf.log.LastLogIndex
		reply.PrevLogTerm = rf.getLastLogTerm()
		return
	}
	```
4. 日志本身非空
   ![日志复制](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE7.png)
   这里又可以分出3种情况，一是所要复制的日志在接收者日志范围之外，则无法复制日志；
   二是在接收者本身日志范围内，且发送的日志条目与接收者本身日志条目无冲突（如图中a、b、c、d），则覆盖接收者对应索引位置的日志
   ```go
   for i, entry := range args.Entries {
		index := args.PrevLogIndex + 1 + i
		if index > rf.log.LastLogIndex {
			rf.log.appendLogs(entry)
		} else if rf.log.getEntry(index).Term != entry.Term {
			ok = false
			*rf.log.getEntry(index) = entry
		}
   }
	if !ok {
		rf.log.LastLogIndex = args.PrevLogIndex + len(args.Entries)
	}
   ```
   > 这里的条目冲突，本人的理解是自己关系，如图中a、b的日志条目都是发送条目的子集，发送条目都是c、d的日志条目的子集。这种情况下，日志的任期可能不一致，Raft中对于这种情况的处理采取覆盖写的方式达成一致性

   三是在接收者本身日志范围内，要复制的日志中有一部分与接收方日志记录中完全相同，此时应该找到完全相同的部分，将其写入接收方日志记录，其余内容舍去（图中e覆盖1-5，f覆盖1-3）
   ```go
    prevIndex := args.PrevLogIndex
	// 寻找相同部分最后截止的索引
	for prevIndex >= rf.log.FirstLogIndex && rf.getEntryTerm(prevIndex) == rf.log.getEntry(args.PrevLogIndex).Term {
		prevIndex--
	}
	reply.FollowerTerm = rf.currentTerm
	reply.Success = false
	if prevIndex >= rf.log.FirstLogIndex {
		reply.PrevLogIndex = prevIndex
		reply.PrevLogTerm = rf.getEntryTerm(prevIndex)
	} else {
		reply.PrevLogIndex = rf.snapshotLastIncludeIndex
		reply.PrevLogTerm = rf.snapshotLastIncludeTerm
	}
   ```

### 日志压缩
日志压缩对于每个节点而言基本上是独立的，即在日志达到一定的大小后将其压缩，其主要过程就是将指定索引之前的日志全部删除，将相关信息记录到快照中，而剩下的部分作为新的本地日志
```go
if rf.log.FirstLogIndex <= index {
	// 只能压缩已经提交的日志
	if index > rf.lastApplied {
		panic("error happening")
	}
	rf.snapshot = snapshot
	rf.snapshotLastIncludeIndex = index
	rf.snapshotLastIncludeTerm = rf.getEntryTerm(index)
	newFirstLogIndex := index + 1
	if newFirstLogIndex <= rf.log.LastLogIndex {
		rf.log.Entries = rf.log.Entries[newFirstLogIndex-rf.log.FirstLogIndex:]
	} else {
		rf.log.LastLogIndex = newFirstLogIndex - 1
		rf.log.Entries = make([]Entry, 0)
	}
	rf.log.FirstLogIndex = newFirstLogIndex
	rf.commitIndex = max(rf.commitIndex, index)
	rf.lastApplied = max(rf.lastApplied, index)
	rf.persist()
	for i := 0; i < len(rf.peers); i++ {
		if i == rf.me {
			continue
		}
		go rf.InstallSnapshot(i)
	}
}
```
此动作的执行在config文件中：
```go
if (m.CommandIndex+1)%SnapShotInterval == 0 {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(m.CommandIndex)
	var xlog []interface{}
	for j := 0; j <= m.CommandIndex; j++ {
		xlog = append(xlog, cfg.logs[i][j])
	}
	e.Encode(xlog)
	rf.Snapshot(m.CommandIndex, w.Bytes())
}
```
#### 日志下载
同样先设计参数
```go
type InstallSnapshotArgs struct {
	Term     int
	LeaderID int

	LastLogIndex int
	LastLogTerm  int
}

type InstallSnapshotReply struct {
	Term    int
	Success bool
}
```
这里需要实现的下载主要指leader对于落后很多的follower或者新加入的节点发送自身的快照，达成日志的同步操作。<br>
对于follower，同样分情况讨论：
1. leader任期过期，拒绝压缩日志
2. follower任期过期，更新follower状态
3. 当合并请求的最大日志索引比当前节点快照所存储的最大日志索引大时，可以合并日志
   ```go
   rf.snapshot = args.Snapshot
   rf.snapshotLastIncludeIndex = args.LastIncludeIndex
   rf.snapshotLastIncludeTerm = args.LastIncludeTerm
   if args.LastIncludeIndex >= rf.log.LastLogIndex {
	  rf.log.Entries = make([]Entry, 0)
	  rf.log.LastLogIndex = args.LastIncludeIndex
   } else {
	  rf.log.Entries = rf.log.Entries[rf.log.getOffset(args.LastIncludeIndex+1):]
   }
   rf.log.FirstLogIndex = args.LastIncludeIndex + 1
   ```

### 日志提交
日志提交
一般来说，在日志复制之后，会进行日志的提交。日志提交是*leader*的动作，当其日志被复制到大多数节点上时，leader会认为这些日志对于整个集群而言都是可执行的，因此会提交这些日志。<br>
```go
if matchIndex <= rf.commitIndex { //commitIndex前的不需要提交
	return
}
// 越界的不能提交
if matchIndex > rf.log.LastLogIndex {
	return
}
if matchIndex < rf.log.FirstLogIndex {
	return
}
// 提交的必须是本任期的日志
if rf.getEntryTerm(matchIndex) != rf.currentTerm {
	return
}

cnt := 1 //自动计算上leader节点的一票
for i := 0; i < len(rf.peers); i++ {
	if i == rf.me {
		continue
	}
	if matchIndex <= rf.trackers[i].matchIndex {
		cnt++
	}
}
if cnt > len(rf.peers)/2 {
	rf.commitIndex = matchIndex
	if rf.commitIndex > rf.log.LastLogIndex {
		panic("")
	}
	return
} else {
	return
}
```

## 其余事项
1. Raft中需要持久化哪些数据？
   本人在实现过程中持久化的数据包括节点的任期、投票对象、日志以及快照信息
2. 几个时间间隔怎么设计？
   Raft中涉及到的时间间隔包括选广播时间、选举超时、平均故障间隔。三者满足如下条件：<br>
   `广播时间（broadcastTime） << 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）` <br>
   其中，广播时间与平均故障间隔不能仅由程序决定，而编程中与算法相关的时间主要有心跳周期与选举超时。心跳周期用于保证leader的权威性，让其余节点检测到leader的状态，而选举超时则为一个选举周期。需要注意的是，如果每个节点的选举周期都同步的话，则投票时很可能发生票数的瓜分，导致一个周期中没有leader。为减少这种情况，一个节点的选举周期通常为一个基本的周期加上一个随机数，使得每个节点尽量不在同一个时间发起选举。<br>
   此次试验中，本人设置选举超时为300ms，而心跳间隔为100ms，即平均每发送两个心跳信号后会进行一次选举。

## 参考资料
Raft论文：https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf<br>
Raft实现思路解析：https://github.com/rfyiamcool/notes
