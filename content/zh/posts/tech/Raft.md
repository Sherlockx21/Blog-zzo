---
keywords:
  - "Raft"
  - "分布式"
title: "MIT6.5840——Lab2A（Leader Election）"
date: 2023-11-02T11:12:11+08:00
lastmod: 2023-11-02T11:12:11+08:00
description: ""
draft: false
author: 椰虂serein
hideToc: false
enableToc: true
enableTocContent: false
tocFolding: false
tocLevels: ["h2", "h3", "h4"]
tags:
  - "MIT"
  - "Raft"
categories: "分布式"
img: ""
---
> 本系列主要作为本人实现MIT6.5840（原MIT6.824）的记录。若有问题，敬请谅解，欢迎指正。<br>

## 传送门
Raft论文：[英文版](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)|[中文版](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md#%E6%91%98%E8%A6%81)
<br>
实验要求：https://pdos.csail.mit.edu/6.824/labs/lab-raft.html
<br>
Raft实现参考实例：https://github.com/hashicorp/raft/tree/main

## 数据结构
数据结构的定义基本参考了论文的图2
![pic2](https://user-images.githubusercontent.com/32640567/116203223-0bbb5680-a76e-11eb-8ccd-4ef3f1006fb3.png)
```go
type Raft struct {
    mu        sync.Mutex          // Lock to protect shared access to this peer's state
    peers     []*labrpc.ClientEnd // RPC end points of all peers
    persister *Persister          // Object to hold this peer's persisted state
    me        int                 // this peer's index into peers[]
    dead      int32               // set by Kill()

    // Your data here (2A, 2B, 2C).
    // Look at the paper's Figure 2 for a description of what
    // state a Raft server must maintain.
    state       State //Raft节点角色，Leader、Candidate、Follower
    currentTerm int //当前的选期
    votedFor    int //投票的对象

    heartbeatTimeout  time.Duration
    electionTimeout   time.Duration
    lastElectionTime  time.Time
    lastHeartbeatTime time.Time
}

type RequestVoteArgs struct {
    // Your data here (2A, 2B).
    Term         int
    CandidateId  int
}

type RequestVoteReply struct {
    // Your data here (2A).
    Term        int
    VoteGranted bool
}

type AppendEntriesArgs struct {
    LeaderTerm   int
    LeaderID     int
    PrevLogIndex int
    PrevLogTerm  int
    Entries      []Entry
    LeaderCommit int
}

type AppendEntriesReply struct {
    FollowerTerm  int  
    Success       bool 
}
```
## 选举过程
整个Lab2A实现领导人选举，主要就是围绕Raft节点的3个基础角色展开
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
            heartbeat := false
            if rf.pastHeartbeatTimeout() {
                heartbeat = true
                rf.resetHeartbeatTimer()
            }
            rf.StartAppendEntries(heartbeat)
        }
        time.Sleep(tickInterval)
    }
}
```
Leader的任务是在心跳到期时发送新的心跳信号，告诉其他节点“我还活着”，不需要在选期中途开始新的选举，同时监控Follower节点的状态。Follower在选期到期后发起新的选举。
```go
func (rf *Raft) StartElection() {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    rf.resetElectionTimer()
    rf.change2Candidate()
    done := false
    votes := 1
    term := rf.currentTerm
    args := RequestVoteArgs{
        Term:rf.currentTerm, 
        CandidateID:rf.me,
    }

    for i, _ := range rf.peers {
        if rf.me == i {
            continue
        }
		// 拉选票
        go func(server int) {
            reply := RequestVoteReply{}
            ok := rf.sendRequestVote(server, &args, &reply)
            // 信息传输出错、未获得选票，选票不增加
            if !ok || !reply.VoteGranted {
                return
            }
            rf.mu.Lock()
            defer rf.mu.Unlock()
            // Follower、Candidate选期落后，刚刚重置为Follower，选票不增加
            if rf.currentTerm > reply.Term {
                return
            }
            // 统计票数
            votes++
            // 已经产生Leader，则停止选举；选票未达到半数以上，继续拉票
            if done || votes <= len(rf.peers)/2 {
                return
            }
            done = true
            // 因为任期的原因转换成了Follower，则不会成为Leader
            if rf.state != Candidate || rf.currentTerm != term {
                return
            }
            rf.state = Leader
            go rf.StartAppendEntries(true) // 立即发送心跳
        }(i)
    }
}
```
### 拉取选票
```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    // Your code here (2A, 2B).
    rf.mu.Lock()
    defer rf.mu.Unlock()
	
    //竞选leader的节点任期小于等于自己的任期，则反对票(为什么等于情况也反对票呢？因为candidate节点在发送requestVote rpc之前会将自己的term+1)
    // Leader选期落后，拒绝给leader投票
    if args.Term < rf.currentTerm {
        reply.VoteGranted = false
        reply.Term = rf.currentTerm
        return
    }
	
    // 选期落后，则同步选期，重置状态为Follower
    if args.Term > rf.currentTerm {
        rf.currentTerm = args.Term
        rf.votedFor = -1
        rf.state = Follower
    }
    reply.Term = rf.currentTerm

    if rf.votedFor == -1 || rf.votedFor == args.CandidateId{
        rf.votedFor = args.CandidateId
        rf.state = Follower
        rf.resetElectionTimer() 
        reply.VoteGranted = true 
    } else { //符合投票条件，但是已经投票给别的节点
        reply.VoteGranted = false
    }
}
```
	


## 心跳检测
Leader Election过程中，心跳信息实际上是一个空的LogEntry。为了区分心跳信息与日志信息，我在这里采用isHeartbeat作为鉴别。
### Leader动作
```go
func (rf *Raft) StartAppendEntries(isHeartbeat bool) {
    args := RequestAppendEntriesArgs{}
    rf.mu.Lock()
    defer rf.mu.Unlock()
    rf.resetElectionTimer()
    args.LeaderTerm = rf.currentTerm
    args.LeaderId = rf.me

    // 发送心跳
    for i, _ := range rf.peers {
        if i == rf.me {
            continue
        }
        go rf.AppendEntries(i, isHeartbeat, &args)
    }
}
```

### Candidate与Follower动作
Candidate与Follower在接收到Leader发送的心跳信息后，根据选期来处理消息。
```go
func (rf *Raft) HandleAppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    rf.mu.Lock()
    reply.Success = true

    // 发现自己的选期比leader的更新，则抛弃leader所传递的消息
    if args.LeaderTerm < rf.currentTerm {
        reply.FollowerTerm = rf.currentTerm
        reply.Success = false
        rf.mu.Unlock()
        return
    }
    rf.mu.Unlock()

    // 承认Leader，将自己的身份转变为Follower
    rf.mu.Lock()
    rf.resetElectionTimer() //重置选举定时器，在选举周期到期前，不会再发起新一轮选举
    rf.state = Follower
    //当选期落后leader，则同步自己的选期
    if args.LeaderTerm > rf.currentTerm {
        rf.votedFor = -1
        rf.currentTerm = args.LeaderTerm
        reply.FollowerTerm = rf.currentTerm
    }
    rf.mu.Unlock()
}
```

## 细节处理
1. 为什么数据结构中缺少了部分论文中存在的变量？
   > 这里只是针对Lab2A所设计的数据结构。Lab2A的测试，仅仅是关于Leader Election的操作，并没有涉及到日志复制相关的操作，因此这里的数据结构会有所缺失。更全面的数据结构会在Lab2B的记录中给出，同时针对AppendEntries和RequestVote函数，也会有相应的改动
2. 在选举过程中，有节点挂了，怎么处理？
   > 首先明确，“挂了”的意思，并不是平时所表达的节点故障，而是节点与其余节点的通信断了，但是节点仍有可能继续运行，也就是“活在自己的世界里”。因此，此时一致性出现的问题，主要就是任期的不一致（在不考虑日志同步的情况下）。明确这点以后，则分两种情况。<br>
   一是主节点挂了，这一过程中，主节点一直“自以为是”的发送心跳信号，同时不会更新自己的选期，在重连以后，节点会在心跳信号的回复中发现从节点的选期（FollowerTerm）比自己更新，因此会更新自己的状态，重新变为Follower。而从节点因为一直收不到心跳信号，也就不会执行resetElectionTimer函数，则选期会过期，过期后从节点则发起新一轮的选举，选期递增，比挂了的主节点更新，后续主节点重连后则会拒绝其信号。
   二是从节点挂了，收不到主节点的心跳信息，因此在重连之前，会不断发生选期超时，不断自增选期，重连后，则会成为选期最新的节点，参与最新一轮的选举。<br>

   > 另一个需要考虑的点，是重连后怎样处理leader发送的心跳信息。其实回应与不回应区别不大。如果回应，则会同上述情况一样使leader重置为follower，然后自己赢得新一轮的选举；如果不回应，follower仍会在自己认为的选期过期后发起新一轮选举，此时会凭借自己的任期优势赢得选举。两种操作结果最终都能实现数据的一致性，因此区别不大。
3. 如果在选举过程中，出现了新的网络分区，出现了多个leader怎么处理？
   > 结论是，不需要特殊处理。多个leader只是暂时的，当网络分区合并后，leaders的选期不会完全相同。而选期更新的leader，则会当选合并后分区的leader，此时，分布式系统的一致性就可以实现了。

