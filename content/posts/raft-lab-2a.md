---
title: "Raft Lab 2a"
date: 2021-06-29T10:42:41+08:00
draft: false
---

Raft实验基于2021的代码。

lab2a 是的目的就是选主，首先需要明确，不使用timer或者ticker，而是使用sleep，这也是实验要求。

代码库已经给我们搭建好了基础框架，我们只需要按照论文的要求来设计和填入代码就能够完成实验，lab2a的大致思路就是：raft协议中的leader定期向follower发送心跳，接受到心跳的follower要及时应答，如果follower没有及时接收到心跳，那么就会触发选主。

### Raft结构体

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

   // 2A leader election and heartbeat
   currentTerm       int       // 当前任期
   state             State     // 节点状态
   votedFor          int       // 投票
   lastHeartBeatTime time.Time // 上次心跳的时间
   heartBeatChan     chan struct{} // 通知接收到心跳的chan
   //TODO 2B and 2C
}
```

Raft 结构体保存了当前节点的状态，已经其他节点的信息。

所以，在Make函数中，我们要返回代表当前节点的raft对象：

```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	// Your initialization code here (2A, 2B, 2C).
	rf.state = Follower
	rf.votedFor = -1
	rf.currentTerm = 0
	rf.heartBeatChan = make(chan struct{})

	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState())

	// start ticker goroutine to start elections
	go rf.electTicker()
	go rf.heartBeatTicker()

	return rf
}
```

可以看到，Make函数做了一些列的初始化操作，并开启了两个后台运行的协程。

### electTicker

```go
func (rf *Raft) electTicker() {
	for rf.killed() == false {
		// 获取随机的选举超时时间
		electTimeout := getElectTimeout()
		// 使用time.Sleep来控制，而不是使用timer，使用timer在系统负载增加的时候可能会延迟。
		time.Sleep(electTimeout)

		if rf.getState() == Leader {
			continue
		}

		duration := time.Since(rf.getHeartbeatTime())
		if duration > electTimeout {
			// 没有收到心跳，发起超时选举
			go rf.election()
		}
	}
}
```

electTicker 方法实现了超时选举，需要注意的是 `getElectTimeout()` 获取的是一个随机的超时时间。

```go
func getElectTimeout() time.Duration {
	ms := 300 + rand.Intn(240)
	return time.Duration(ms) * time.Millisecond
}
```

### heartBeatTicker

```go
func (rf *Raft) heartBeatTicker() {
	for rf.killed() == false {
		time.Sleep(heartBeatInterval)
	  // 不是leader， 不能发送心跳
		if rf.getState() != Leader {
			continue
		}
		for i := range rf.peers {
			if rf.me == i {
				continue
			}
			//DPrintf("[server-%d] heartBeatTicker to peer[%d]\n", rf.me, i)
			go rf.sendHeartBeat(i)
		}
	}
}
```

以上两个函数就是实现超时选举的关键，其他rpc请求只需要按照论文所要求的的翻译出来即可。

### 如何赢得选票？

在candidate进行选举的时候，会向所有的peer发送`RequestVote` 这个PRC请求，如果收到的投票大于等于`len(rf.peers)/2 + 1`  , 也就是一半以上，那么candidate就会变成leader。

```go
// 发送请求
for i := range rf.peers {
		if i == rf.me {
			continue
		}
		reply := &RequestVoteReply{}
		go func(server int) {
			if !rf.sendRequestVote(server, args, reply) {
				//DPrintf("[server-%d] sendRequestVote failed rpc to peer[%d]\n", rf.me, server)
				notifyChan <- struct{}{}
				return
			}
			rf.mu.Lock()
			defer rf.mu.Unlock()

			if startTerm != rf.currentTerm {
				return
			}

			if reply.Term > rf.currentTerm {
				rf.becomeFollower(reply.Term)
			}

			if reply.VoteGranted {
				grantVoteCount++
			}
			notifyChan <- struct{}{}
		}(i)
	}
	
	// for循环获取结果，并添加超时处理
	for {
		select {
		case <-notifyChan:
			rf.mu.Lock()
			if rf.currentTerm != startTerm {
				rf.mu.Unlock()
				return
			}
			if rf.state != Candidate {
				rf.mu.Unlock()
				return
			}

			if grantVoteCount >= rf.getMajority() {
				rf.becomeLeader()
				rf.mu.Unlock()
				return
			}

			rf.mu.Unlock()
		case <-time.After(getElectTimeout()):
			// 退出
			DPrintf("[server-%d] elect timeout, state[%d]\n", rf.me, rf.getState())
			return
		}
	}
```

lab2a的实现还是很简单的，但是在选举超时处理，和心跳方面需要下点功夫，不然三个测试会过不了哦。

```go
➜  raft git:(feat/raft_2a) ✗ go test -run 2A -race
Test (2A): initial election ...
  ... Passed --   3.0  3   46    6142    0
Test (2A): election after network failure ...
  ... Passed --   4.5  3   96    8501    0
Test (2A): multiple elections ...
  ... Passed --   5.5  7  480   42904    0
PASS
ok      6.824/raft      13.397s
```

