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

1. 拉票者（发送者）任期过期

   ```go
   if args.Term < rf.currentTerm {
   	reply.VoteGranted = false
   	return
   }
   ```

2. 

### 发送日志