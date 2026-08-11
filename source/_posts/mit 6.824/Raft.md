---
title: Raft
author: setKing
date: 2026-06-20 11:33:00 +0800
categories: [MIT 6.824]
tags: [论文, 分布式, 原理]
pin: true
math: true
mermaid: true
---

<center>Raft</center>

### 核心理念

Raft是一种用于实现复制状态机一致性的共识算法，通过管理复制日志，使集群中的多个节点按照相同顺序执行相同操作，从而保证集群中状态机状态的一致性。。Raft算法由Leader、Follower和Candidate组成。Raft首先会选举一个特定的Leader，然后将日志复制管理职责完全赋予该Leader，进而实现“共识”。 Leader接收来自客户端的日志条目， 在其他服务器上进行复制，并通知各节点何时可以安全地将日志条目应用于状态机。 Follower是Raft集群中的接收者，会将Leader广播的日志接收到本地，然后在日志按顺序提交后重放，Candidate在节点在规定时间内没有收到有效Leader消息时触发选举机制，Candidate由Follower在选举超时时转换而来，通过向其他节点请求投票竞争Leader。竞选过程中只有获得票数在集群中过半会成为Leader

### 关键细节

#### Raft共识算法

下图总结了Raft算法：

![](.\img\raft.jpg)

下图列出了该算法的关键属性：

![](.\img\Raft_key.jpg)

Raft将共识问题分解为三个相对独立的子问题：

- **Leader选举：** 现有Leader崩溃时必须重新选举一个Leader；
- **日志复制：** Leader必须接受来自客户端的日志条目，并将其复制到集群中其他节点，并强迫其他节点接受自己的数据；
- **安全性：** Raft关键的安全属性指的是图3中的状态机安全：如果任何节点已将指定日志条目应用于状态机，则其他节点不得在同一个位置(index)应用不同的命令。5.4节描述了Raft时如何保证此属性的；其中会对5.2节中描述的选举机制施加额外限制。

Raft集群的部署节点推荐大于2的奇数个节点，偶数节点会降低可用性，增加无法形成多数派的概率。

**选举**

Raft将时间划分为连续递增的逻辑周期，每个周期称为一个任期（Term），用连续整数表示Term，term在Raft中起到逻辑时钟的作用。每个term都以选举开始，选举中一个或多个Candidate试图成为Leader。如果一个Candidate在选举中获胜，则在该term其余时间内担任Leader。Raft确保在一个term中至多有一个Leader。

Raft中，借助term，节点能够检测到过时的信息，例如过时的Leader。每个节点存储一个当前term的编号，随着时间单调递增。节点间每次通信时，都会交换当前term信息。如果一个节点的当前term比另一个小，则该节点会将其term更新为较大的值。如果Candidate或Leader发现其term已过期，则会立即将自己的角色重置为Follower 。如果节点收到带有过时term的请求，则会直接拒绝处理。

Raft服务器通过远程过程调用(RPC)进行通信

Raft结构体组成：

```
type ApplyMsg struct {
	CommandValid  bool        // 表示该 ApplyMsg 是否包含一个新提交的日志条目
	Command       interface{} // 该 ApplyMsg 包含的命令
	CommandIndex  int         // 该 ApplyMsg 包含的命令在日志中的索引
	SnapshotValid bool        // 表示该 ApplyMsg 是否包含一个快照
	Snapshot      []byte      // 该 ApplyMsg 包含的快照数据
	SnapshotTerm  int         // 快照中包含的最后一个日志条目的任期
	SnapshotIndex int         // 快照中包含的最后一个日志条目的索引
}

// LogEntry 日志条目：包含任期号和命令。
type LogEntry struct {
	Term    int         // 该日志条目的任期号
	Command interface{} // 该日志条目的命令
}

// 实现单个 Raft 节点的 Go 对象。
type Raft struct {
	mu                sync.Mutex          // 用于保护对该节点共享状态的并发访问
	peers             []*labrpc.ClientEnd // 所有节点的 RPC 端点
	persister         *Persister          // 用于保存该节点持久化状态的对象
	me                int                 // 该节点在 peers[] 中的索引
	dead              int32               // 由 Kill() 设置
	currentTerm       int                 // 当前任期
	votedFor          int                 // 投票给了谁
	log               []LogEntry          // 日志
	commitIndex       int                 // 已提交的日志索引
	lastApplied       int                 // 已应用到状态机的日志索引
	electionTimer     *time.Timer         // 选举定时器
	state             string              // 节点状态：Follower、Candidate 或 Leader
	nextIndex         []int               // 领导者为每个跟随者维护的下一个日志索引
	matchIndex        []int               // 领导者为每个跟随者维护的已复制日志索引
	ApplyMsgChan      chan ApplyMsg       // 用于向服务（或测试器）发送 ApplyMsg 的通道
	applyMu           sync.Mutex          // 保护 applyCommitted 中的 channel 写入顺序
	lastIncludedIndex int                 // 快照中包含的最后一个日志索引
	lastIncludedTerm  int                 // 快照中包含的最后一个日志条目的任期
	// 在此添加你的数据（2A, 2B, 2C）。
	// 参考论文的 Figure 2，了解 Raft 服务器
	// 必须维护的状态。
}
```

Candidate会在选举期间发起RequestVote RPC调用来获取投票：

```
// RequestVote RPC 请求参数结构体。
// 字段名必须以大写字母开头！
type RequestVoteArgs struct {
	Term         int // 候选人的任期号
	CandidateId  int // 请求选票的候选人ID（服务器编号）
	LastLogIndex int // 候选人最后日志条目的索引值
	LastLogTerm  int // 候选人最后日志条目的任期号
}

// RequestVote RPC 响应结构体。
// 字段名必须以大写字母开头！
type RequestVoteReply struct {
	Term        int  // 当前任期号
	VoteGranted bool // 是否投票给该候选人
}

// RequestVote RPC 处理函数。
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// 拒绝任期过旧的请求，并返回当前任期
	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
		return
	}
	// 领导者任期过旧，更新任期并转变为跟随者
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = "Follower"
		rf.votedFor = -1
		rf.persist()
		rf.resetElectionTimer()
	}
	// 拒绝日志不够新的候选人
	myLastLogIdx := rf.logLen() - 1
	myLastLogTerm := rf.log[len(rf.log)-1].Term
	if args.LastLogTerm < myLastLogTerm || (args.LastLogTerm == myLastLogTerm && args.LastLogIndex < myLastLogIdx) {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
		return
	}
	// 投票给第一个请求投票的候选人，或者已经投票给该候选人
	if rf.votedFor == -1 || rf.votedFor == args.CandidateId {
		reply.VoteGranted = true
		reply.Term = rf.currentTerm
		rf.votedFor = args.CandidateId
		rf.resetElectionTimer()
		rf.persist()
	} else {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
	}

}
```

Raft使用心跳机制来触发Leader选举。节点启动时，初始角色是Follower。只要Follower节点能持续接收到来自Leader或Candidate的有效RPC，其就会维持在Follower角色。 Leader会定期向所有Follower发送心跳（不包含日志条目的 AppendEntries RPC），用以维持自己的"权威"。 如果Follower在选举超时时间内没有收到任何通信，就会认定当前没有有效的Leader，那么它就会发起新一轮选举来选出新的Leader。

以下是选举的实现：

```
// electionLoop 跟随者/候选人/领导者主循环，统一管理状态转换。
// 只由 Make() 启动一个 goroutine，heartbeatLoop 同步调用，不产生新 goroutine。
func (rf *Raft) electionLoop() {
	for !rf.killed() {
		<-rf.electionTimer.C

		rf.mu.Lock()
		// 已是 Leader（timer 可能在 heartbeatLoop 运行期间触发）
		if rf.state == "Leader" {
			rf.mu.Unlock()
			rf.heartbeatLoop()
			rf.mu.Lock()
			rf.resetElectionTimer()
			rf.mu.Unlock()
			continue
		}

		// 转变为候选人，发起选举
		rf.state = "Candidate"
		rf.currentTerm++
		rf.votedFor = rf.me
		currentTerm := rf.currentTerm
		lastLogIndex := rf.logLen() - 1
		lastLogTerm := rf.log[len(rf.log)-1].Term
		rf.resetElectionTimer()
		rf.persist()
		rf.mu.Unlock()

		// 并发广播 RequestVote
		votes := 1                      // 投给自己
		majority := len(rf.peers)/2 + 1 // 过半数
		var votesMu sync.Mutex
		becameLeader := false // 是否已经成为领导者
		var wg sync.WaitGroup
		for i := 0; i < len(rf.peers); i++ {
			if i == rf.me {
				continue
			}
			wg.Add(1)
			go func(server int) {
				defer wg.Done()
				args := &RequestVoteArgs{
					Term:         currentTerm,
					CandidateId:  rf.me,
					LastLogIndex: lastLogIndex,
					LastLogTerm:  lastLogTerm,
				}
				reply := &RequestVoteReply{}
				if !rf.sendRequestVote(server, args, reply) {
					return
				}

				rf.mu.Lock()
				// 当前任期过旧，更新任期并转变为跟随者
				if reply.Term > rf.currentTerm {
					rf.currentTerm = reply.Term
					rf.state = "Follower"
					rf.votedFor = -1
					rf.resetElectionTimer()
					rf.persist()
					rf.mu.Unlock()
					return
				}
				// 不是候选人或任期不匹配，忽略回复
				if rf.state != "Candidate" || rf.currentTerm != currentTerm {
					rf.mu.Unlock()
					return
				}
				// 投票给了自己，统计投票结果
				if reply.VoteGranted {
					votesMu.Lock()
					votes++
					if votes >= majority && !becameLeader {
						becameLeader = true
						votesMu.Unlock()
						rf.state = "Leader"
						for j := 0; j < len(rf.peers); j++ {
							rf.nextIndex[j] = rf.logLen()
							rf.matchIndex[j] = rf.lastIncludedIndex
						}
					} else {
						votesMu.Unlock()
					}
				}
				rf.mu.Unlock()
			}(i)
		}
		done := make(chan struct{})
		go func() {
			wg.Wait()
			close(done)
		}()
		select {
		case <-done:
		case <-rf.electionTimer.C:
		}
		// 排空 channel 中的残留定时器值，防止下次循环读到过期事件
	drain:
		for {
			select {
			case <-rf.electionTimer.C:
			default:
				break drain
			}
		}

		rf.mu.Lock()
		if rf.state == "Leader" {
			rf.mu.Unlock()
			rf.heartbeatLoop()
			rf.mu.Lock()
		}
		rf.resetElectionTimer()
		rf.mu.Unlock()
	}
}
```

为了开启新一轮选举，Follower会增加它的当前term，并转换为Candidate角色。然后它为自己投票，并向集群中的每个节点并行发出RequestVote请求。节点会停留在Candidate角色，直到发生下列三种情况之一：

(a) 它赢得了选举

(b) 另一台服务器宣布自己为Leader

(c) 一段时间过去了但没有选出Leader。

Candidate在获得集群中超过半数选票时，就赢得了选举。每个节点在一个给定term中最多只投一票，采用先到先得原则。Candidate一旦赢得选举，就会成为Leader。然后它会向所有其他节点发送心跳消息以建立权威并抑制新的选举。

等待投票时，Candidate可能会收到另一个声称是Leader节点的AppendEntries RPC。当该Leader的Term大于等于本节点的Term时，Candidate将承认该Leader，并将自己重置为Follower。小于的话会拒绝该RPC并继续处于Candidate角色。

```
   // 拒绝任期过旧的请求，并返回当前任期
	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
		return
	}
	// 领导者任期过旧，更新任期并转变为跟随者
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = "Follower"
		rf.votedFor = -1
		rf.persist()
		rf.resetElectionTimer()
	}
```

如果Candidate既没有赢得选举也没有输掉选举，每个Candidate都会超时并开始新一轮选举，方式是递增term值并发起另一轮RequestVote RPC。为了避免无限重复选举，Raft使用随机的选举超时来确保上述选票分散场景的发生频率较低：

```
// resetElectionTimer 重置选举定时器为随机超时时间。
func (rf *Raft) resetElectionTimer() {
	ms := 400 + rand.Intn(400)
	if rf.electionTimer == nil {
		rf.electionTimer = time.NewTimer(time.Duration(ms) * time.Millisecond)
	} else {
		if !rf.electionTimer.Stop() {
			select {
			case <-rf.electionTimer.C:
			default:
			}
		}
		rf.electionTimer.Reset(time.Duration(ms) * time.Millisecond)
	}
}
```

每个Candidate在选举开始时重置其随机选举超时计时器，并等待该超时时才开始下一轮选举；这降低了新选举中再次发生投票分散的可能性。

**日志复制**

一旦选举出Leader，它就开始接收客户端请求。每个客户端请求都包含一个要由复制状态机执行的指令。 Leader将该指令作为新日志条目追加到本地日志，然后并发地向其他节点发出AppendEntries RPC以复制该条目。 当日志条目安全复制并提交后，Leader会将其应用于本地状态机并将执行结果返回给客户端。 如果Follower崩溃或运行缓慢，或者网络数据包丢失，那么Leader将持续重试AppendEntries RPC（即使已经将响应发给了客户端），直到所有Follower最终存储了所有日志条目为止。

```
// AppendEntries RPC 响应。
// 字段名必须以大写字母开头！
type AppendEntriesReply struct {
	Term    int  // 当前任期号，用于更新自身任期
	Success bool // 如果跟随者的日志与 prevLogIndex 和 prevLogTerm 匹配，则为 true
	XTerm   int  // 领导者任期过旧时，返回跟随者的当前任期
	XIndex  int  // 领导者任期过旧时，返回跟随者日志中第一个与 XTerm 匹配的索引
	XLen    int  // 领导者任期过旧时，返回跟随者日志的长度
}

// AppendEntries RPC 处理函数。
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	// 传入的任期过旧，拒绝并返回当前任期
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		return
	}
	// 传入的任期大于当前任期，更新任期并转变为跟随者
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		reply.Term = rf.currentTerm
		rf.state = "Follower"
		rf.votedFor = -1
		rf.resetElectionTimer()
		rf.persist()
	}
	// PrevLogIndex 在快照覆盖范围内，follower 已无此条目
	if args.PrevLogIndex < rf.lastIncludedIndex {
		reply.Term = rf.currentTerm
		reply.Success = false
		reply.XLen = rf.lastIncludedIndex
		rf.resetElectionTimer()
		return
	}
	// 传入的日志索引或任期不匹配，拒绝并返回冲突信息
	// PrevLogIndex 是逻辑索引，需要转换成物理索引来访问 rf.log
	if args.PrevLogIndex >= rf.logLen() ||
		rf.log[rf.logIdx(args.PrevLogIndex)].Term != args.PrevLogTerm {
		reply.Term = rf.currentTerm
		reply.Success = false
		if args.PrevLogIndex >= rf.logLen() {
			reply.XLen = rf.logLen()
		} else {
			physIdx := rf.logIdx(args.PrevLogIndex)
			reply.XTerm = rf.log[physIdx].Term
			reply.XIndex = args.PrevLogIndex
			for reply.XIndex > rf.lastIncludedIndex && rf.log[rf.logIdx(reply.XIndex)-1].Term == reply.XTerm {
				reply.XIndex--
			}
		}
		rf.resetElectionTimer()
		return
	}
	// 日志匹配，接受并追加新日志条目
	for i, entry := range args.Entries {
		logicalIdx := args.PrevLogIndex + 1 + i
		if logicalIdx < rf.logLen() {
			if rf.log[rf.logIdx(logicalIdx)].Term != entry.Term {
				rf.log = rf.log[:rf.logIdx(logicalIdx)]

				rf.log = append(rf.log, entry)
			}
		} else {
			rf.log = append(rf.log, entry)
		}
	}
	rf.persist()
	// 更新 commitIndex 并应用已提交的日志条目
	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = min(args.LeaderCommit, rf.logLen()-1)

		rf.applyCommitted()
	}
	reply.Success = true
	rf.resetElectionTimer()
	reply.Term = rf.currentTerm

}
// 向服务器发送 AppendEntries RPC。
func (rf *Raft) sendAppendEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	return ok
}
```

Leader决定何时将日志条目应用到状态机是安全的；我们称满足条件的条目为“已提交(committed)”。Raft保证提交的日志条目会被持久化存储且最终会被所有可用的状态机执行。一旦创建条目的Leader在多数节点上完成了复制。就会提交该条目。这也会提交Leader日志中的所有前驱条目，包括由之前的Leader创建的条目。Leader会维护其感知到的已提交的最大index，并将其通过AppendEntries RPC（包括心跳消息）传给其他节点。一旦Follower感知到某日志条目已提交，它就在本地状态机中按日志顺序执行对应的条目。

已提交但未应用的日志条目应用到状态机：

```
// applyCommitted 将已提交但未应用的日志条目应用到状态机，并通过 ApplyMsgChan 发送 ApplyMsg。
// 调用方必须持有 rf.mu，返回时仍持有 rf.mu。
func (rf *Raft) applyCommitted() {
	if rf.commitIndex <= rf.lastApplied {
		return
	}
	msg := make([]ApplyMsg, 0, rf.commitIndex-rf.lastApplied)
	for i := rf.lastApplied + 1; i <= rf.commitIndex; i++ {
		msg = append(msg, ApplyMsg{
			CommandValid: true,
			Command:      rf.log[rf.logIdx(i)].Command,
			CommandIndex: i,
		})
	}
	rf.lastApplied = rf.commitIndex
	rf.mu.Unlock()

	rf.applyMu.Lock()
	for _, m := range msg {
		rf.ApplyMsgChan <- m
	}
	rf.applyMu.Unlock()

	rf.mu.Lock()
}
```

为了“日志可对齐”属性，Raft维护以下属性：

- 如果两个不同日志中的条目具有相同的index和term，则它们所存储指令也相同。
- 如果两个不同日志中的条目具有相同的index和term，则两日志中所有前驱条目都是相同的

第一个性质源于Leader在一个term中最多只创建一个具有给定index的日志条目，而日志条目日志中的位置永远不会变。第二个性质是由AppendEntries RPC执行的一个简单的一致性检查保证的。

在Raft算法中，Leader通过日志匹配机制，使Follower日志最终与Leader保持一致。这意味着Follower日志中的冲突条目将被来自Leader的日志条目所覆盖。

为了使Follower的日志与自己的保持一致，Leader必须找到两个日志能达成一致的最新日志条目，然后删除Follower日志中该条目之后的所有条目，并向Follower发送该条目之后的所有Leader的条目。

**安全性**

Raft不会通过计数被复制的节点数来提交前一个term的日志条目。只有来自Leader当前term的日志条目的提交才通过计数的方式进行；一旦当前term中的条目被以这种方式提交，则根据"日志可对齐"属性，所有以前的条目都会间接被提交。

**Follower和Candidate的崩溃**

如果一个Follower或Candidate崩溃了，则后续发送给它们的RequestVote RPC和AppendEntries RPC请求都将失败。Raft通过无限重试来处理这些故障； 如果发生崩溃的节点重新启动， 则该请求被正常处理。 如果在发送响应之前崩溃， 则它在重新启动后将再次收到相同的RPC。而Raft RPC是幂等的， 因此不会有任何影响。

**时间和可用性**

对Raft的要求之一是，其安全性不能依赖于时间：系统不能仅仅因为某些事件发生得比预期更快或更慢就产生不正确的结果。然而，可用性（系统及时响应客户端的能力）必然是基于时间的。

Leader选举是Raft中最依赖时间的阶段。只要系统满足以下的**时间要求**，Raft就能够选举并维护一个稳定的Leader：

broadcastTime ≪ electionTimeout ≪ MTBF

在不等式中，broadcastTime是节点向集群中的每个节点并行发送RPC并接收到响应所需的平均时间；electionTimeout是选举超时时间；MTBF是单个节点故障之间的平均间隔时间。

broadcastTime比electionTimeout小几个数量级，以便Leader可以可靠地发送心跳消息，用以抑制Candidate启动选举；该不等式也是"脑裂"的case不会出现的必要条件之一。electionTimeout应比MTBF小几个数量级，以便系统可以稳定地向前推进。broadcastTime和MTBF是底层系统的固有属性，而electionTimeout则是我们必须要去选择的。

领导者定期广播心跳：

```
// heartbeatLoop 领导者定期广播心跳（空的 AppendEntries）。
func (rf *Raft) heartbeatLoop() {
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()
	for !rf.killed() {
		rf.mu.Lock()
		if rf.state != "Leader" {
			rf.mu.Unlock()
			return
		}
		rf.broadcastAppendEntries()
		rf.mu.Unlock()

		<-ticker.C
	}
}

// broadcastAppendEntries 向所有跟随者发送 AppendEntries 或 InstallSnapshot（持有 rf.mu 时调用）。
func (rf *Raft) broadcastAppendEntries() {
	for i := 0; i < len(rf.peers); i++ {
		if i == rf.me {
			continue
		}
		// follower 落后到日志已被截断的区域，发送快照
		if rf.nextIndex[i] <= rf.lastIncludedIndex {
			snapArgs := &InstallSnapshot{
				Term:              rf.currentTerm,
				LeaderId:          rf.me,
				LastIncludedIndex: rf.lastIncludedIndex,
				LastIncludedTerm:  rf.lastIncludedTerm,
				Data:              rf.persister.ReadSnapshot(),
				Done:              true,
			}
			go func(server int) {
				reply := &InstallSnapshotReply{}
				if !rf.sendInstallSnapshot(server, snapArgs, reply) {
					return
				}
				rf.mu.Lock()
				if reply.Term > rf.currentTerm {
					rf.currentTerm = reply.Term
					rf.state = "Follower"
					rf.votedFor = -1
					rf.persist()
					rf.resetElectionTimer()
				} else {
					rf.nextIndex[server] = snapArgs.LastIncludedIndex + 1
					rf.matchIndex[server] = snapArgs.LastIncludedIndex
				}
				rf.mu.Unlock()
			}(i)
			continue
		}

		if rf.nextIndex[i] > rf.logLen() {
			rf.nextIndex[i] = rf.logLen()
		}
		prevLogIdx := rf.nextIndex[i] - 1
		args := &AppendEntriesArgs{
			Term:         rf.currentTerm,
			LeaderId:     rf.me,
			PrevLogIndex: prevLogIdx,
			PrevLogTerm:  rf.log[rf.logIdx(prevLogIdx)].Term,
			Entries:      rf.log[rf.logIdx(rf.nextIndex[i]):],
			LeaderCommit: rf.commitIndex,
		}
		go func(server int) {
			reply := &AppendEntriesReply{}
			if !rf.sendAppendEntries(server, args, reply) {
				return
			}
			rf.mu.Lock()
			// 领导者任期过旧，更新任期并转变为跟随者
			if reply.Term > rf.currentTerm {
				rf.currentTerm = reply.Term
				rf.state = "Follower"
				rf.votedFor = -1
				rf.persist()
				rf.resetElectionTimer()
				rf.mu.Unlock()
				return
			}
			// 日志更新成功
			if reply.Success {
				rf.matchIndex[server] = args.PrevLogIndex + len(args.Entries)

				rf.nextIndex[server] = rf.matchIndex[server] + 1
				// 检查是否有新的日志条目被提交
				lastPhys := len(rf.log) - 1
				for N := lastPhys; N > rf.logIdx(rf.commitIndex); N-- {
					logicalN := rf.lastIncludedIndex + N
					count := 1 // 包括自己
					for j := 0; j < len(rf.peers); j++ {
						if j != rf.me && rf.matchIndex[j] >= logicalN && rf.log[N].Term == rf.currentTerm {
							count++
						}
					}
					if count >= len(rf.peers)/2+1 {
						rf.commitIndex = logicalN

						rf.applyCommitted()
						break
					}
				}
			} else {
				if reply.XTerm != 0 {
					next := prevLogIdx
					for next > rf.lastIncludedIndex && rf.log[rf.logIdx(next)-1].Term == reply.XTerm {
						next--
					}
					if next == prevLogIdx {
						next = reply.XIndex
					}
					rf.nextIndex[server] = next
				} else {
					rf.nextIndex[server] = reply.XLen
				}
			}
			rf.mu.Unlock()
		}(i)
	}
}
```

#### 集群成员的变更

而在Raft中，为应对集群成员变更，集群首先切换到一个称为“联合共识”的过渡配置；一旦该配置被提交，系统就会转换为新的配置。

- 日志条目会复制到两配置中加一起的所有节点；
- 两种配置中的任一节点都可以当选Leader；
- 决策（选举和日志条目的提交）需要同时得到旧配置和新配置的多数选票；

“联合共识”允许在不牺牲安全性的前提下，在不同的时间变更各节点的配置。此外，联合共识还允许群集在整个配置变更期间继续为客户端提供服务。

集群配置以日志副本的特殊条目的形式在集群中存储和传输；配置变更的流程是当Leader收到一个新旧配置更改的请求时，会生成一个C(旧,新)的联合配置存储为日志条目并使用之前描述的机制来将其复制到其他节点。一旦某个节点将其新的配置条目添加到其日志中，它就会一直使用该配置来进行后续决策。Leader将根据基于C(旧,新)的规则来判定C(旧,新)何时可提交。如果Leader崩溃了，新Leader可能会在C(旧)或C(旧,新)中产生，具体取决于胜选的Candidate是否收到了C(旧,新)。在此期间，无论如何都不能基于C(新)单方面做出决策。

一旦C(旧,新)被提交，C(旧)和C(新)都不能在未经对方批准的情况下做出决策，并且领导者完整性确保了仅带有C(旧,新)日志条目的节点才能被选举为Leader。

> 配置只要被节点接收，就会立即生效。

配置变更有三个问题需要解决:

1. 新的节点可能最初不存储任何日志条目（会导致可能需要很长的时间才能跟上进度）。解决办法是Raft在配置更改之前引入了一个额外阶段（学习者），在这个阶段中，新节点以**非投票**成员的身份加入集群（Leader会将日志条目复制给它们，但在判定是否集齐多数派回应时不会考虑这些节点）。当新节点追上了群集中的其他节点后，就配置变更过程就可以按照上面描述的方式进行处理了。
2. 原集群Leader可能不在新配置之中。在这种情况下，解决办法是一旦Leader提交了新的日志条目C(新)，它就会被降级（重置为Follower）。
3. 已删除的节点（不在C(新)中的节点）可能会干扰集群。不过这些节点不会接收到心跳信息，所以它们会超时并开始新的选举。

为了解决这个问题，节点在认为当前存在Leader时会忽略RequestVote RPC。

#### 日志压缩

在正常运行期间，Raft日志会随客户端请求的增多而增长。

Raft 中使用快照，每个节点都可以独立地创建快照，其中只包含其日志中已提交的部分条目。状态机的大部分工作是将其当前状态写入快照。Raft还在快照中包含少量元数据：lastIncludedIndex是被快照替换掉的最后一个日志条目（即状态机应用的最后一个条目）的index，lastIncludedTerm是该条目的term。这些信息保留下来的目的是支持快照后第一个AppendEntries RPC的一致性检查，因为过程中需要上一个日志条目的term和index。为了支持集群成员变更，快照还包括到lastIncludedIndex为止的最新配置。节点一旦完成了快照的写入，它就可以删除所有index小于等于lastIncludedIndex的日志条目，以及所有老版本快照。

Leader在某些场景下必须向落后的Follower发送快照。 比如下一个需要发送给落后Follower的日志条目已被删除（因为Leader完成了快照）。

Leader使用一个称为InstallSnapshot的新类型RPC将快照发送给过于落后的Follower。当一个Follower通过该RPC接收到一个快照时，它必须判断如何处理本地现有的日志条目。通常情况下，快照会包含接收方日志中不存在的新信息。在这种情况下，Follower会丢弃其整个日志，都用快照代替，日志中也可能包含未提交且与快照冲突的条目。否则，由于重传或错误，追随者接收到对应其日志前缀的快照，那么快照对应的日志条目将被删除，但位于快照之后的条目仍然有效并且必须保留。

![](.\img\installSnapshot.jpg)

这种快照方法偏离了Raft的强Leader原则，因为Follower可以在无需Leader信息的前提下进行快照。但是这个原则的违背是有道理的。虽然拥有一个Leader有助于避免在达成共识过程中产生冲突，但当进行快照时已经达成了共识，因此没有冲突需要Leader去协调。数据仍然只从Leader流向Follower，只是Follower也有了重组自己数据的能力。

以下是快照的实现：

```
// 持久化数据的序列化（供 persist 和 Snapshot 共用）
func (rf *Raft) persistData() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	e.Encode(rf.lastIncludedIndex)
	e.Encode(rf.lastIncludedTerm)
	return w.Bytes()
}
// 将 Raft 的持久化状态保存到稳定存储中，
// 以便在崩溃和重启后可以恢复。
func (rf *Raft) persist() {
	rf.persister.SaveRaftState(rf.persistData())
}

// 恢复之前持久化的状态。
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // 没有状态数据，初始启动？
		return
	}
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var currentTerm int
	var votedFor int
	var lastIncludedIndex int
	var lastIncludedTerm int
	var log []LogEntry
	if d.Decode(&currentTerm) != nil ||
		d.Decode(&votedFor) != nil ||
		d.Decode(&log) != nil ||
		d.Decode(&lastIncludedIndex) != nil ||
		d.Decode(&lastIncludedTerm) != nil {

		// 错误处理...
	} else {
		rf.currentTerm = currentTerm
		rf.votedFor = votedFor
		rf.log = log
		rf.lastIncludedIndex = lastIncludedIndex
		rf.lastIncludedTerm = lastIncludedTerm
	}
}
// InstallSnapshot RPC 请求参数结构体。
type InstallSnapshot struct {
	Term              int    // 领导者的任期号
	LeaderId          int    // 领导者的 ID
	LastIncludedIndex int    // 快照中包含的最后一个日志索引
	LastIncludedTerm  int    // 快照中包含的最后一个日志条目的任期
	Offset            int    // 快照数据的偏移量
	Data              []byte // 快照数据
	Done              bool   // 是否为最后一个快照数据块
}
type InstallSnapshotReply struct {
	Term int // 当前任期号，用于更新自身任期
}

func (rf *Raft) Snapshot(index int, snapshot []byte) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if index > rf.logLen()-1 {
		index = rf.logLen() - 1
	}
	if index <= rf.lastIncludedIndex {
		return
	}
	rf.log = rf.log[rf.logIdx(index):]
	rf.lastIncludedIndex = index
	rf.lastIncludedTerm = rf.log[0].Term

	// 截断日志后调整 nextIndex/matchIndex，避免越界
	for i := range rf.peers {
		if rf.nextIndex[i] < rf.lastIncludedIndex+1 {
			rf.nextIndex[i] = rf.lastIncludedIndex + 1
		}
		if rf.nextIndex[i] > rf.logLen() {
			rf.nextIndex[i] = rf.logLen()
		}
		if rf.matchIndex[i] < rf.lastIncludedIndex {
			rf.matchIndex[i] = rf.lastIncludedIndex
		}
	}
	rf.persister.SaveStateAndSnapshot(rf.persistData(), snapshot)
}

func (rf *Raft) InstallSnapshot(args *InstallSnapshot, reply *InstallSnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	if args.Term < rf.currentTerm {
		return
	}
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = "Follower"
		rf.votedFor = -1
		rf.persist()
		rf.resetElectionTimer()
	}

	if args.LastIncludedIndex <= rf.lastIncludedIndex {
		return
	}

	rf.lastIncludedIndex = args.LastIncludedIndex
	rf.lastIncludedTerm = args.LastIncludedTerm
	rf.log = []LogEntry{{Term: args.LastIncludedTerm}}

	if args.LastIncludedIndex > rf.commitIndex {
		rf.commitIndex = args.LastIncludedIndex
	}
	if args.LastIncludedIndex > rf.lastApplied {
		rf.lastApplied = args.LastIncludedIndex
	}

	rf.persister.SaveStateAndSnapshot(rf.persistData(), args.Data)

	rf.applyMu.Lock()
	rf.ApplyMsgChan <- ApplyMsg{
		SnapshotValid: true,
		Snapshot:      args.Data,
		SnapshotTerm:  args.LastIncludedTerm,
		SnapshotIndex: args.LastIncludedIndex,
	}
	rf.applyMu.Unlock()
}

func (rf *Raft) sendInstallSnapshot(server int, args *InstallSnapshot, reply *InstallSnapshotReply) bool {
	ok := rf.peers[server].Call("Raft.InstallSnapshot", args, reply)
	return ok
}
```

#### 客户端交互

Raft客户端将所有请求发送到Leader。当客户端启动时会随机连接到一个节点。如果客户端第一次选择的不是Leader，那么该节点将会拒绝客户端的请求，并返回所感知到的最新Leader信息（AppendEntries RPC包括Leader的网络地址）。如果Leader崩溃，客户端请求会超时；然后客户端会再次尝试连接到随机的节点。

#### 总结

Raft 为系统构建提供了更好的基础。Raft的性能与其他共识算法（如 Paxos）类似。性能评估中最重要的场景是Leader对日志条目的复制。Raft通过最少的消息数来提升这一场景的效率。Raft可以通过批量处理和管道操作来提高性能
