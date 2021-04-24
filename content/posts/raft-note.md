---
title: "Raft Note"
date: 2020-09-01T14:38:08+08:00
draft: true
---

本文记录了在学习 Raft 算法的过程中的一些笔记和个人理解。

Raft 是近年来非常受欢迎的一致性算法，其目的就是为了在多个节点达成共识，允许一组机器像一个整体一样工作，即使一些机器出现故障、网络出现分区也能继续工作下去。

- **目标**：解决分布式系统中，对于单个数据值如何达成一致的问题
- **前提**：这些系统构建在普通的商用机器上，是不可靠的，随时有挂掉的风险
- **容错**：必须接受部分节点离线或者死机，比如 raft 容忍 (n-1)/2的成员挂掉

### 初探 Raft

典型的写入请求流程，client 发送 request 到 leader 节点，共识模块将用户请求写入本地 log中，然后广播 entry 到所有的 follower 节点，当 quorum 超过半数确认收到数据后，leader commit 这条 entry，然后 apply 到本地状态机 (fsm) 中，最后返回给 client。那么 follower 什么时候 apply 这条记录呢？leader 下次发送请求或者 heartbeat 心跳时，会携带 leader commited index，follower 就可以 commit 这个 index 之前的所有 entry。

### Raft 的特点

相对于其他共识算法，Raft 有很多特点，使得更加容易理解和工程实现

- Strong leader：所有的数据流向都是 leader 传播到 follower，所以 raft 实现的服务是一个 `CP` 系统，在 leader 挂掉的时间，不能提供服务。
- Leader election：随机的超时时间，follower 收不到 leader 的心跳后，不会同一时间发起选举请求，避免选票被瓜分 split vote。
- Membership changes `joint consensus` 用于解决成员变更问题

为了实现共识，Raft 将工作划分了几个任务，`Leader election`，`Log replication`，`Safety`，`Membership changes` 和 `Log Compaction`。

#### 1. Leader election

Raft 里面有 term 的概念，当 Leader 的角色不发生变化时，term 维持不变，直到 Leader 挂掉或者有 Candidate 触发选举，term 单调递增。

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-term.png)

从图中可以看到，一个任期 term 的时间被划分两部分：election 和 normal operation，但是也有例外，比如 `t3` 时刻只有 election 时间，可能此时发生了 split vote 现象，选票被瓜分了。

集群中有三种角色：Follower，Candidate，Leader。

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-role.png)

初始时所有节点状态为 Follower，在一定时间内没有收到任何 AppendEntries 或是 Heartbeat 后，变成 Candidate 同时投自己一票，并且 term+1，然后并行发送 RequestVote 到其他节点。之后会有三种情况：

1. 同一任期 term，一个节点只能做一次投票，按照**先来先得**的原则，Follower 会把选票投给 Candidate。收到过半数的投票后，Candidate 会成为新的 Leader，然后发送心跳给所有的 Follower。
2. 已有其他节点成为 Leader，**或是 term 低于其他节点**，那么当前节点都会被拒绝，并退回 Follower。
3. 选举超时，继续将 term+1



### 复制状态机

一致性算法是在复制状态机的背景下提出的。在这种方法中，一组服务器上的状态机产生相同状态的副本，并且在一些机器宕机的情况下也能够继续运行。复制状态机本广泛的运用在分布式系统中，用来解决很多容错的问题。

![复制状态机](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-%E5%9B%BE1.png)

> 图 1 ：复制状态机的结构。一致性算法管理着来自客户端指令的复制日志。状态机从日志中处理相同顺序的相同指令，所以产生的结果也是相同的。

复制状态机通常都是基于**复制日志**实现的。如图 1，每一个服务器存储一个包含一系列指令的日志，并且按照日志的顺序进行执行。每一个日志都按照相同的顺序包含相同的指令，所以每一个服务器都执行相同的指令序列。因为每个状态机都是确定的，每一次执行操作都产生相同的状态和同样的序列。

