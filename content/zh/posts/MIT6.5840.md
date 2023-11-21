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

Raft论文：

Raft实现参考实例：

## 基本思路概述

MIT6.5840主要实现分布式一致性算法（Raft）。在实验任务中，主要分为领导人选举、日志复制、持久化、日志压缩四个部分。不过本文主要以Leader、Candidate、Follower三个角色为切入点来讲解思路，故四个部分不会分的特别明确，望谅解。

### 节点角色

Raft中的分布式节点主要有三个角色，Leader、Candidate、Follower。因此，此处定义数据结构

```go
type State int
const (
	Follower State = iota
	Candidate
	Leader
)
```

Follower与Candidate的任务，主要是在任期结束之后发起新的选举；Leader的任务，则是不断向其余节点发送心跳或者传输日志。实验中，Raft算法的执行通过**ticker**函数实现

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
			// 心跳定时器过期发送心跳，否则发送日志
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

主要的选举过程如下

```go
for i, _ := range rf.peers {
		if rf.me == i {
			continue
		}
		// 拉选票
		go func(server int) {
			var reply RequestVoteReply
			ok := rf.sendRequestVote(server, &args, &reply) //发送请求
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
	}
```

Q1：`if rf.state != Candidate || rf.currentTerm != term` 判断的是哪种情况？

A1：首先要明确，当一个节点断联的时候，节点本身不一定是无法运行的状态，可能只是RPC通信错误导致的。此时，如果一个节点断联，则会有两种情况。若节点为Leader，则会不断发送心跳，但不会收到回复，也就不会更新任期。在重联后，会因为自身任期旧而失去Leader身份重置为Follower。若节点为Follower，在断联时接收不到心跳，故自身的选举计时器会不断超时，导致自己不停发起选举，重联后自己任期一定为最新，可以成为Leader。因为在拉票之前，节点会将自己的身份转变为Candidate，这里的条件就是判断节点是否在选举过程中断联，如果断联，重置前不会参与选举。

#### 拉票

拉票是整个选举的核心内容，决定了每个节点的角色。整个过程主要会出现下列几种情况：

1. Candidate（发送者）任期过期，拒绝投票

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
3. 两者任期相同，此时要求Candidate的日志和接收者一样新（具体表现为Candidate最后一条日志任期比接收者更大或者在任期相同的情况下日志索引更大）
   ```go
   update := false
	update = update || args.LastLogTerm > rf.getLastLogTerm()
	update = update || args.LastLogTerm == rf.getLastLogTerm() && args.LastLogIndex >= rf.log.LastLogIndex

	if (rf.votedFor == -1 || rf.votedFor == args.CandidateId) && update {
		//竞选任期大于自身任期，则更新自身任期，并转为follower
		rf.votedFor = args.CandidateId
		rf.state = Follower
		rf.resetElectionTimer() //自己的票已经投出,转为follower
		rf.persist()
	} else {
		reply.VoteGranted = false
	}
	```
> 这里的日志比较其实可以类比为文档，选取文档时，版本（日志的任期）更高的文档往往比版本低的更贴近于技术的更新，而在版本相同时，我们更趋向于选取内容更全（日志的索引）的文档作为参考。
## Leader的任务
Candidate与Follower节点的功能相对简单，接下来就是Leader的工作。Leader与Follower的交互主要包括发送心跳与日志消息。
### 发送心跳
心跳在分布式系统中的作用就是保证节点的活跃。在Raft算法中，心跳本质上是通过一个空的日志消息实现的。
```go
// 并发的向各个节点发送消息
for i, _ := range rf.peers {
	if i == rf.me {
		continue
	}
	go rf.AppendEntries(i, heart)
}
```
在AppendEntries函数中，首先要保证发送消息这个动作是原子性的，即不能被别的动作所打扰。这里利用go中的锁来实现。
```go
rf.mu.Lock()
if rf.state != Leader {
	rf.mu.Unlock() //必须解锁，否则会造成死锁
	return
}
reply := AppendEntriesReply{}
args := AppendEntriesArgs{}
args.LeaderTerm = rf.currentTerm
args.LeaderID = rf.me
rf.mu.Unlock()
```
```go
rf.mu.Lock()
if rf.state != Leader {
	rf.mu.Unlock()
	return
}
// Follower任期落后，重置Follower状态，Leader不做特殊处理
if reply.FollowerTerm < rf.currentTerm {
	rf.mu.Unlock()
	return
}
// Leader任期落后于Follower，因此Leader需转变为Follower并跟新任期
if reply.FollowerTerm > rf.currentTerm {
	rf.votedFor = -1
	rf.state = Follower
	rf.currentTerm = reply.FollowerTerm
}
rf.mu.Unlock()
```
对于接收者而言，处理心跳只有接受和拒绝两种方式

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

### 发送日志消息

发送日志消息的过程与上述发送心跳的实现基本一致，不过需要多一些对日志的讨论

```go
type AppendEntriesArgs struct {
	LeaderTerm   int //leader 任期
	LeaderID     int 
	PrevLogIndex int //传递的日志的前一条日志下标
	PrevLogTerm  int 
	Entries      []Entry //日志信息
	LeaderCommit int //leader提交的日志下标
}

type AppendEntriesReply struct {
	FollowerTerm int //follower任期
	Success      bool
	PrevLogIndex int 
	PrevLogTerm  int
}
```

接收者对于日志消息的过程主要如下：

`接收消息`-->`判断leader任期`-->`重置选举计时器`-->`判断自身任期`-->`更新日志`

```go
if args.LeaderTerm < rf.currentTerm {
	reply.Success = false
	return	
}
rf.resetElectionTimer()
rf.state = Follower

if args.LeaderTerm > rf.currentTerm {
	rf.votedFor = -1
	rf.currentTerm = args.LeaderTerm
	reply.FollowerTerm = rf.currentTerm
}
```

接下来讨论日志的复制：

1. 需要复制的日志在自身日志范围外，直接拒绝复制日志

   ```go
   if args.PrevLogIndex+1 < rf.log.FirstLogIndex || args.PrevLogIndex > rf.log.LastLogIndex {
   	reply.FollowerTerm = rf.currentTerm
   	reply.Success = false
   	reply.PrevLogIndex = rf.log.LastLogIndex
   	reply.PrevLogTerm = rf.getLastLogTerm()
   }
   ```

2. 

