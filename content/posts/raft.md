---
title: "Raft"
date: 2021-06-27T23:12:50+08:00
draft: true
---

# 1. Raft 简介

Raft 是近年来非常受欢迎的共识算法，其目的就是为了在多个节点达成共识，允许一组机器像一个整体一样工作，即使一些机器出现故障、网络出现分区也能继续工作下去。这使得 Raft 在构建可靠的大型软件系统中起着关键作用。

相比于其他共识算法，Raft 有以下特点：

- Strong leader：Raft 的日志只能由 leader 复制给其他 follower
- Leader election：Raft 使用随机的超时时间进行 Leader 选择
- Membership changes：Raft 改变集群中的服务器集的机制使用了一种新的联合共识方法，在这种方法中，**两个不同的配置的多数会在过渡期间重叠**。这使得集群在改变配置时能够继续正常运行。

Raft 假设：

- 服务器可能会宕机、会停止运行过段时间再恢复，但是**非拜占庭的**（即它的行为是非恶意的，不会篡改数据等）；
- 网络通信会中断，消息可能会丢失、延迟或乱序；可能会网络分区；

我们将从 6 个部分解释 Raft：

1. Leader 选举；
2. 日志复制；
3. 安全性；
4. 配置变更：如何在集群中增加或删除节点；

# 2. 复制状态机

一致性算法是在复制状态机的背景下提出的。在这种方法中，一组服务器上的状态机产生相同状态的副本，并且在一些机器宕机的情况下也能够继续运行。复制状态机本广泛的运用在分布式系统中，用来解决很多容错的问题。

![https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-%E5%9B%BE1.png](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-%E5%9B%BE1.png)

图1 复制状态机的结构。一致性算法管理着来自客户端指令的复制日志。状态机从日志中处理相同顺序的相同指令，所以产生的结果也是相同的。

复制状态机通常都是基于**复制日志**实现的。如图1，每一个服务器存储一个包含一系列指令的日志，并且按照日志的顺序进行执行。每一个日志都按照相同的顺序包含相同的指令，所以每一个服务器都执行相同的指令序列。因为每个状态机都是确定的，每一次执行操作都产生相同的状态和同样的序列。

# 3. 相关术语

## **服务器状态**

服务器在任意时间只能处于以下三种状态之一，在正常操作中，只有一个 Leader，其他所有的服务器都是 Follower：

- Leader：处理所有客户端请求、日志复制。同一时刻最多只能有一个可行的 Leader；
- Follower：完全被动的（不发送 RPC，只响应收到的 RPC）——大多数服务器在大多数情况下处于此状态；
- Candidate：用来选举新的 Leader，处于 Leader 和 Follower 之间的暂时状态；

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-server-state.png)

## 任期

Raft 把时间分为任意长度的一个个的任期（`Term`），任期用连续的整数来编号。每个任期都是以 Candidate 发起选举操作开始的，每个任期只能有一位或者没有 Leader 诞生。

一个正常的任期至多有一个 Leader，通常分为两部分：

- 任期开始时的选举过程；
- 正常运行的部分；

有些任期可能没有选出 Leader，这时候 Candidate 会立即进入下一个任期，再次尝试选出一个 Leader。

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-term.png)

每个节点维护一个 currentTerm 变量，表示系统中当前任期。currentTerm 必须持久化存储，以便在服务器宕机重启时将其恢复。

### **RPC**

Raft 服务器的通信通过两个 RPC 调用：

- `RequestVote`：用于选举；
- `AppendEntries`：用于复制 log 和发送心跳；

# 4. Leader 选举

Raft 使用心跳机制来触发 Leader 选举。

1. 当服务器启动的时候，都是 Follower 状态，只要 Follower 能够在计时器超时前接收到 Candidate 或者 Leader 的 RPC 调用，那么就会保持 Follower 状态；
2. Leader 周期性的发送心跳给所有的 Follower来 maintain their authority；
3. 在计时器超时后，Follower 会变成 Candidate，并且发起新一轮的选举；

## 选举过程

选举从 Follower 的计时器超时开始：

1. Follower 自增 `currentTime` ，并且变为 Candidate 状态；
2. Candidate 给自己投票，设置 `voteFor` 为自己，并且并行的给其他服务器发起 `RequestVote` RPC请求；
3. Candidate 会保持当前状态，等待响应，直到以下三种情况发生：
   - 赢得选举，变了 Leader
   - 集群中已经有了 Leader，变为 Follower
   - 选票被瓜分，currentTerm++，发起新一轮选举

下面分别讨论这三种情况：

### 赢得选举

如果 Candidate 从大多数服务器中获得当前任期的投票，那么 Candidate 赢得了选举，变为 Leader。然后它向所有服务器发送心跳来建立权威，也能够阻止新一轮的选举。

为了保证选举的安全性，选举需要遵循以下规则：

- 每台服务器在给定的 Term 内最多只能为一名 Candidate 投票，以先到先得的方式进行；
- 只有获得超过半数节点的选票才能成为 Leader，少数服从多数的原则确保最多只有一名 Candidate 能在某一 Term 内赢得选举；

### 回退到 Follower

在等待投票期间，Candidate 有可能收到其他服务器的声明自己是 Leader 的 `AppendEntries` RPC 请求。如果 Leader 的 Term 大于等于 currentTerm，那么 Candidate 回退到 Follower 状态；反之，Candidate 会拒绝当前 RCP。

### 保持 Candidate 状态

第三个可能的结果是，一个 Candidate 既没有赢得选举，也没有输掉选举：如果许多 Follower 同时成为 Candidate，票数可能被瓜分，因此没有候选人获得多数。当这种情况发生时，每个候选人都会超时，并通过增加其 Term 和启动新一轮的 `RequestVote` RPC 来开始新的选举。然而，**如果没有额外的措施，瓜分的投票可能会无限地重复。**

Raft 使用随机的选举超时时间来确保极少出现选票瓜分，以及出现之后也能够很快的被解决掉。为了防止一开始就出现分裂投票，选举超时是从一个固定的时间间隔中随机选择的（例如，150-300ms）。

# 5. 日志复制

Leader 确认之后，就会开始接受客户端的请求。当 Leader 接收到客户端的请求之后，会把追加一条新日志，并且通过 `AppendEntries` RPC 把日志复制给其它服务器。

## 日志的组织方式

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-log-structure.png)

每个服务器都有一份自己的日志副本，但是 Follower 的日志最终会和 Leader 日志保持一致。每个日志条目都存储了一条命令以及其他信息，它的具体结构如下：

```go
type Entry struct {
	Term int64
	Index int64
	Command Command
	...
}
```

其中必定包含的是：

- Index：日志条目的位置
- Term：日志条目被创建时的任期号
- 命令

## 日志提交

在 Leader 把一条日志复制到大多数服务器之后，日志就被提交，并且 Raft 算法保证已经提交的日志是持久化的，并且最终会被所有可用的状态机执行。注意：提交操作同样也会提交领导者日志中所有之前的条目，包括之前领导者创建的条目。日志被提交之后，才能被安全地应用到状态机。

日志的复制提交流程如下：

1. 客户端向 Leader 发送命令；
2. Leader 先把命令追加到自己的日志中，然后 Leader 并行地向其它节点发送 `AppendEntries` RPC，等待响应；
3. 收到超过半数节点的响应，则认为新的日志记录是被提交的：
   - Leader 将命令应用到状态机，然后向客户端返回响应
   - 此外，一旦 Leader 知道一条记录被提交了，将在后续的 `AppendEntries` RPC 中通知已经提交记录的 Followers
   - Follower 将已提交的命令传给自己的状态机
4. 如果 Follower 宕机/超时：Leader 将反复尝试发送 RPC；

## 日志匹配（Log Matching）

Raft 的日志机制，使得在不同服务器上的日志之间保持高度的一致性。Raft 同样为日志维持了以下两个属性：

1. 如果在不同服务器间的两个日志有相同的 `index` 和 `term`，那么他们存储了相同的命令；
2. 如果在不同服务器间的两个日志有相同的 `index` 和 `term`，那么日志中的所有前面的条目都是相同的。

### AppendEntries 一致性检查

为了维持上面的两个属性，Follower 在收到 Leader 的 `AppendEntries` RPC 时会进行一致性检查。

Leader 为每个 Follower 维护了一个 `nextIndex` 变量，表示下一个将要发送给 Follower 的日志的 index。当 Leader就任时，它将所有 Follower 的 nextIndex 初始化为其日志中最后一个索引之后的索引。

当 `AppendEntries` RPC 发送：

- 对于每个 `AppendEntries` RPC 包含新日志**之前那条日志的**索引(`prevLogIndex`)和任期(`prevLogTerm` )；
- Follower 检查自己的 index 和 term 是否与 `prevLogIndex` 和 `prevLogTerm` 匹配，匹配则接收该日志；否则拒绝；
- Leader 的 RPC 被拒绝后，他会回退 nextIndex 到之前的某个点，然后重试，知道成功为止。

# 6. 安全性

为了保证状态机的一致性，**Raft 保证之前任期已提交的所有日志都会出现在之后的新 Leader 的日志中，一旦 Leader 的状态机执行了一条日志里的命令，必须确保其它状态机在同样索引的位置不会执行不同的命令。**前面的章节中，并没有对选举做出任何限制，所以选举过程存在安全性问题。例如，当领导者提交几个日志条目时，一个跟随者可能无法使用，然后它可能被选为领导者，并用新的条目覆盖这些条目；结果，不同的状态机可能执行不同的命令序列，导致了状态机的不一致。

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-commited-log.png)

## 选举限制

Raft 使用选举来阻止某些 Candidate 成为 Leader，除非他们已经包含了已提交的全部日志。Raft 通过 `RequestVote` RPC 调用来实现选举限制：

- RPC 包括 Candidate 的日志信息；
- 如果投票者自己的日志比 Candidate 的日志`更新`，则投票者会拒绝投票。

Raft 通过比较两个服务器日志中的最后一条日志条目的 index 和 term 来判断哪个服务器的日志更新：

1. 如果两个日志条目有不同的 term，那么 term 大的日志条目的服务器更新；
2. 如果两个日志条目有相同的 term，那么 index 更大的日志条目的服务器更新；

## 提交以前任期的日志

当新的 Leader 上任时，如果对以前任期的日志复制和提交方式不加限制，那么可能会导致一些不一致性。

我们来看个例子：

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/raft-log-bad-commit-rule.png)

一个时间序列显示了为什么Leader不能使用 old term 的日志条目来决定是否提交日志。

上图描述了旧的日志就算被存储在大多数服务器，也能够被未来的新的 Leader 给覆盖掉，这就导致了不一致性。

(a) S1 是 Leader，复制了 Index2 的日志到 S2；

(b) S1 崩溃，S5 成为 Leader（选票来自S3,S4,自己），并且在 Index2 产生了的日志；

(c) S5 崩溃，S1 成为了 Leader，接着，他把之前`未提交的 term2 的日志`复制到了大多数节点，下面可能发生 d,e两种情况

(d) S1 崩溃，S5成为 Leader，然后他把之前`未提交的 term3 的日志`复制到了大多数节点，并且用 term3 的日志覆盖了 S1 在 (c) 阶段复制的日志；

(e) S1 把当前任期 4 的日志复制到大多数节点，并且提交，这也间接提交了之前任期的日志，这时 S5 便能成为 Leader，所有之前任期的日志也被提交。

为了消除这个问题，Raft 提出：

1. Leader 不会通过副本计数来提交之前任期的日志，只有 Leader 当前任期的日志才能通过副本计数来提交，一旦当前任期的日志被提交，那么之前任期的日志也就会通过之前描述的日志匹配规则间接提交。
2. **Leader 不能单独提交以前任期的日志，必须通过当前任期的日志提交来间接提交以前任期的日志**

结合上面新的选举和日志提交规则，我们可以保证 Raft 的安全性。具体的论证可以看 Raft 论文的 5.4.3节。
