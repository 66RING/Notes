
# 2A

## abstract

raft共识算法实现可抽象为"leader选举"、"日志复制"。本质上也是通过冗余来保证数据的安全性，而raft算法则通过一些简单抽象让这个过程具有强一致性的同时还便于理解。简单的说leader向follower广播日志来冗余数据，而follower中会通过心跳、任期等信息的判断来保证一致性。

- leader节点
	* 接收客户端指令然后向其他节点广播追加操作日志，当半数以上节点做好备份后向客户端响应
	* 定期向其他节点发送心跳信号heartbeat以告知leader的主导权，当follower节点长时间收不到心跳信号时将发起选举
- follower节点
	* 做冗余备份，等待leader节点通知客户端的指令然后保存到本地日志中
	* follower节点长期收不到心跳信号时(心跳超时)将发起选举，变为candidate节点，请求成为新leader
- candidate节点
	* 处于选举状态的节点，当收到半数以上的赞成票时成为leader节点，收到半数以上反对票时成为leader
- 每个节点都要对投票请求做出响应


## tinykv raft框架

raft节点创建根据配置文件初始化节点自身的状态，然后初始化在该节点视角内的其他节点的同步情况`r.Prs`。`Progress.Match`表示节点所认为的其他节点已经匹配的日志编号，`Progress.Next`表示节点所认为而下一此日志追加应该追加的日志编号。

```go
func newRaft(c *Config) *Raft {
	if err := c.validate(); err != nil {
		panic(err.Error())
	}
	// Your Code Here (2A).

	// 在持久化中获取状态
	hardState, _, err := c.Storage.InitialState()
	if err != nil {
		panic(err)
	}

	// 新建raft对象
	r := &Raft{
		id:               c.ID,
		Term:             hardState.Term,
		Vote:             hardState.Vote,
		RaftLog:          newLog(c.Storage),
		Prs:              make(map[uint64]*Progress),
		State:            StateFollower,
		votes:            make(map[uint64]bool),
		Lead:             None,
		heartbeatTimeout: c.HeartbeatTick,
		electionTimeout:  c.ElectionTick,
	}

	// 初始化所有peers
	// 新节点不知道其他的节点的日志信息，故
	// match为0，next为节点最新的Index
	lastIndex := r.RaftLog.LastIndex()
	for _, prs := range c.peers {
		if prs == r.id {
			r.Prs[prs] = &Progress{Match: lastIndex, Next: lastIndex + 1}
		} else {
			r.Prs[prs] = &Progress{Next: lastIndex + 1}
		}
	}
	return r
}
```

tinkvk项目中使用`tick()`函数来做逻辑时钟驱动，leader节点会根据此间歇发送心跳信号，follower节点会根据此计算是否心跳超时。

```go
// 逻辑时钟计时器: 超时计时， 心跳间隔计时
func (r *Raft) tick() {
	// Your Code Here (2A).
	switch r.State {
	// leader心跳间隔计时器
	case StateLeader:
		tickHeartbeat(r)
	// follwer和candidate都是超时计时器
	default:
		tickElection(r)
	}
}
```

`Step(m pb.Message)`方法用于表现系统的执行，通过`Step`发送一条信息从而让指定节点做相应的处理。

```go
func (r *Raft) Step(m pb.Message) error {
	// Your Code Here (2A).
	switch r.State {
	case StateFollower:
		if err := stepFollower(r, m); err != nil {
			panic(err)
		}
	case StateCandidate:
		if err := stepCandidate(r, m); err != nil {
			panic(err)
		}
	case StateLeader:
		if err := stepLeader(r, m); err != nil {
			panic(err)
		}
	}
	return nil
}
```

每个节点根据收到的指令(`m pb.Message`)执行下一步操作，如回应选举请求、执行日志追加等。以follower节点为例，不同指令做不同处理：

```go
func stepFollower(r *Raft, m pb.Message) error {
	switch m.MsgType {
	// 追加日志
	case pb.MessageType_MsgAppend:
		r.becomeFollower(m.Term, m.From)
		r.handleAppendEntries(m)
	// 处理心跳信号
	case pb.MessageType_MsgHeartbeat:
		r.handleHeartbeat(m)
	case pb.MessageType_MsgPropose:
	// hup对于follower来说是心跳中断，
	// 将发起新一轮竞选
	case pb.MessageType_MsgHup:
		r.Campaign()
	// 处理投票请求
	case pb.MessageType_MsgRequestVote:
		r.HandleVoteReq(m)
	}

	return nil
}
```

其中`pb.Message`定义了如下指令类型(`m.MsgType`)和含义

- `MessageType_MsgAppend`: 表示m是日志追加信号，节点要将日志追加到本地
- `MessageType_MsgHeartbeat`: 表示m是心跳信号，收到说明leader存活
- `MessageType_MsgHeartbeatResponse`: 表示m是心跳响应信号，leader根据信息内容更新其他节点
- `MessageType_MsgPropose`: 表示m是客户端请求，leader先要追加本地日志，然后向其他节点广播
- `MessageType_MsgHup`: 表示m是心跳超时信号，leader失活，follower要发起竞选
- `MessageType_MsgRequestVote`: 表示m是投票请求信号，根据情况投赞成或反对
- `MessageType_MsgRequestVoteResponse`: 表示m是投票请求回复信号，统计赞成和反对的信息以确实变成follower还是leader
- `MessageType_MsgBeat`: 表示m是心跳间隔到期，向其他节点发送心跳信号


## 2AA: leader选举"循环"

### leader

leader节点在选举循环部分需要实现如下功能

- 定期向其他节点发生心跳信号，确认leader存活
- 当收到心跳信号时根据携带的任期号判断。如果是新leader的心跳信号，说明当前leader的"过时"了的，要成为新leader的follower(如当前leader掉线后重新上线缺不知已有新leader的情况)。当收到的信号是旧leader的信号，说明旧leader刚复活，则不变成follower
- 回复其他节点的投票请求。其他节点刚超时发起竞选请求时leader正好恢复上线的情况。这时的leader可能是"过时"的leader，所以如果竞选的节点任期更高则投赞成票

一个tick后，根据心跳间隔计数，当心跳间隔到期时向其他节点发送心跳信号宣誓存活。

```go
tick() -> tickHeartbeat()

func tickHeartbeat(r *Raft) {
	// 心跳间隔计数器
	r.heartbeatElapsed++

	// 一个心跳信号周期到期
	if r.heartbeatElapsed >= r.heartbeatTimeout {
		r.heartbeatElapsed = 0
		// MessageType_MsgBeat，仅用于leader向其他节点发送心跳信号
		if err := r.Step(pb.Message{From: r.id, MsgType: pb.MessageType_MsgBeat}); err != nil {
			panic("heartbeatTimeout panic: " + err.Error())
		}
	}
```

因为可能存在老leader复活时新leader已经上任的情况，所以如果leader节点收到心跳信号，而且心跳信号的任期比自己的更新则自己不再有资格当leader变为follower节点：

```go
func stepLeader(r *Raft, m pb.Message) error {
	switch m.MsgType {
		case pb.MessageType_MsgHeartbeat:
		if m.Term > r.Term {
			r.becomeFollower(m.Term, m.From)
		}
		...
	}
}
```

如果收到的是投票请求信号，当发起请求的节点任期比自己大，日志比自己新而且自己没有投票则投赞成票。需要注意的是，可能多轮竞选都没成功，在新竞选发起时要清除投票信息

```go
func (r *Raft) HandleVoteReq(m pb.Message) {
	// 向发送者发送投票回复的内容
	reply := pb.Message{
		MsgType:              pb.MessageType_MsgRequestVoteResponse,
		To:                   m.From,
		From:                 r.id,
		Term: 			      r.Term,
		Reject: 			  true,
	}

	// 检查是否已经投票
	// 任期比自己小将拒绝
	if m.Term < r.Term {
		reply.Term = r.Term
		reply.Reject = true
	} else  {
		// 如果任期大，说明已经是新一轮竞选了
		// 重置投票信息
		if m.Term > r.Term {
			r.becomeFollower(m.Term, None)
		}
		lastIndex := r.RaftLog.LastIndex()
		logTerm, _ := r.RaftLog.Term(lastIndex)
		// 至少日志要一样新(通过判断logTerm和Index)才能投同意
		if (r.Vote == None || r.Vote == m.From) {
			// 如果日志任期一样，比较index是否比较新
			// 如果日志任期较高，同意
			if (logTerm == m.LogTerm && lastIndex <= m.Index) || 
				logTerm < m.LogTerm {
				reply.Reject = false
				r.Vote = m.From
			}
		}
	}
	r.send(reply)
}
```

### follower

follower节点在选举循环部分需要实现如下功能

- 收到心跳信号时重置超时时间
- 当长时间没有收到心跳信号，触发心跳超时。说明当前leader可能失活，变为candidate节点发起投票请求，竞选leader
- 收到其他节点的投票请求时，对方的任期比自己大，日志比自己新而且自己没有投票则投赞成票，否则投反对票

follower收到心跳信号说明leader存活，重置心跳超时计数器。为了防止一直发送多个节点同时超时、同时发起竞选从而导致竞选失败，重置心跳超时计数器时要重置为随机数。

```go
func (r *Raft) handleHeartbeat(m pb.Message) {
	// Your Code Here (2A).
	// 重置超时时间
	r.electionElapsed = 0 - globalRand.Intn(r.electionTimeout)
	msg := pb.Message{
		MsgType: pb.MessageType_MsgHeartbeatResponse,
		From:    r.id,
		To:      m.From,
		Term:    r.Term,
		Commit:  r.RaftLog.committed,
		Index:   r.RaftLog.stabled,
	}
	if m.Term < r.Term {
		msg.Reject = true
	} else {
		r.Lead = m.From
		msg.Reject = false
	}
	r.send(msg)
}
```

如果时间流逝(`tick()`)，长时间没能收到心跳信号，则说明leader失活了，follower要发起竞选成为新的leader领导整个系统。发出`MessageType_MsgHup`信号让节点得知心跳超时，发起选举。

```go
func (r *Raft) tick() {
	// Your Code Here (2A).
	// follwer和candidate都超时
	default:
		tickElection(r)
	}
}

func tickElection(r *Raft) {
	r.electionElapsed++
	// 选举超时则发起选举
	if r.electionElapsed >= r.electionTimeout {
		r.electionElapsed = 0
		// 发送超时信号，进入竞选
		if err := r.Step(pb.Message{From: r.id, MsgType: pb.MessageType_MsgHup}); err != nil {
			panic("electionTimeout panic: " + err.Error())
		}
	}
}

stepFollower() -> Campaign()

// 发起竞选
// 1. 自身成为candidate
// 2. 并行向所有peer发送投票请求
func (r *Raft) Campaign() {
	// 1. 自身成为候选人
	r.becomeCandidate()
	// 如果只有1个节点则立刻成为leader
	// 因为不会向自己发送VoteReq，没法触发VoteReq中的处理
	if len(r.Prs) == 1 {
		r.becomeLeader()
		return
	}

	// 2. 向所有peers发送投票请求
	lastIndex := r.RaftLog.LastIndex()
	logTerm, _ := r.RaftLog.Term(lastIndex)
	for prs := range r.Prs {
		if prs != r.id {
			r.send(pb.Message{
				MsgType:              pb.MessageType_MsgRequestVote,
				To:                   prs,
				From:                 r.id,
				Term:                 r.Term,
				LogTerm: 			  logTerm,
				Index: 			  	  lastIndex,
			})
		}
	}
}
```

同时，如果一个follower节点收到其他节点的投票请求需要判断是否同意当选。

```go
case pb.MessageType_MsgRequestVote:
	r.HandleVoteReq(m)
}
```

### candidate

candidate节点在选举循环部分需要实现如下功能

- follower节点心跳超时时将变为candidate节点，然后发起投票请求，竞选leader
- 收到心跳信号时，可能是新leader发出的也可能是旧leader发出的，判断如果是新leader则变为follower
- candidate收到投票响应，统计赞成票，当半数以上节点同意时则成为leader，当半数以上节点反对时成为follower
- candidate也响应投票请求，如果对方的任期比自己大，日志比自己新而且自己没有投票则投赞成票，否则投反对票

当一个节点发起竞选后改节点将变为candidate节点：

```go
// 发起竞选
// 1. 自身成为candidate
// 2. 并行向所有peer发送投票请求
func (r *Raft) Campaign() {
	// 1. 自身成为候选人
	r.becomeCandidate()
	// 如果只有1个节点则立刻成为leader
	// 因为不会向自己发送VoteReq
	// 保证奇数个节点??
	if len(r.Prs) == 1 {
		r.becomeLeader()
		return
	}

	// 2. 向所有peers发送投票请求
	lastIndex := r.RaftLog.LastIndex()
	logTerm, _ := r.RaftLog.Term(lastIndex)
	for prs := range r.Prs {
		if prs != r.id {
			r.send(pb.Message{
				MsgType:              pb.MessageType_MsgRequestVote,
				To:                   prs,
				From:                 r.id,
				Term:                 r.Term,
				LogTerm: 			  logTerm,
				Index: 			  	  lastIndex,
			})
		}
	}
}
```

如果candidate收到了心跳信号，说明有leader存活。但是leader可能是真正的新leader也可能是复活的老leader，判断后做出响应：

```go
case pb.MessageType_MsgHeartbeat:
	// 心跳信号可能是复生的老leader发出的
	if m.Term > r.Term {
		r.becomeFollower(m.Term, m.From)
	}
	r.handleHeartbeat(m)
```


因为选举超时时间的随机的，candidate同样处理投票请求。对方的任期比自己大，日志比自己新而且自己没有投票则投赞成票，否则投反对票


```go
case pb.MessageType_MsgRequestVote:
	r.HandleVoteReq(m)
```

candidate根据收到的投票响应判断自身是否可以成为leader。当半数以上节点同意时则成为leader，当半数以上节点反对时成为follower

```go
case pb.MessageType_MsgRequestVoteResponse:
	// 记录选票信息
	r.votes[m.From] = !m.Reject
	total := len(r.Prs)

	// 记录赞成票
	if m.Reject == false {
		r.granted++
	} else {
		r.rejected++
	}

	// 当半数以上同意时立刻成为leader
	if r.granted > total/2 {
		r.becomeLeader()
	} else if r.rejected > total/2 {
		r.becomeFollower(r.Term, m.From)
	}
```

## 2AB: 日志复制"循环"

日志复制循环的主要流程是：

1. leader收到客户端信息，自己本地记录后，根据所记录的节点日志信息(`Prs[i].Match`, `Prs[i].Next`)向其他节点广播日志复制请求
2. follower收到日志请求判断一致性情况后再进一步操作
	* 如果leader任期低`m.Term < r.Term`，(m代表消息发送方，这里就是leader。r表示节点本身，这里就是follower)，直接拒绝
	* 如果follower的日志"过多"，即`r.LastIndex > m.Index`的情况，如果`r.Term(m.Index)!=m.LogTerm`follower日志和leader记录一致，则可以覆盖地追加。
		+ 优化：如果follower记录和leader记录一致，可以不进行删除再添加，直接跳过
	* 如果follower的日志缺失`r.LastIndex < m.Index`或follower日志与leader中的记录冲突(`m.LogTerm!=r.LogTerm`的情况)
		+ follower查找之前的日志，当`r.LogTerm < m.LogTerm`时，找到可能的不冲突日志，向leader反馈
		+ leader收到follower反馈，同样找到可能不冲突的日志`r.LogTerm < m.LogTerm`，尝试向follower发送同步
3. 当日志信息同步完成后，向leader反馈。leader更新commit信息，向客户端响应


### 数据结构

日志的表示`Raftlog`

```go
// RaftLog manage the log entries, its struct look like:
//
//  snapshot/first.....applied....committed....stabled.....last
//  --------|------------------------------------------------|
//                            log entries
//
type RaftLog struct {
	storage Storage
	committed uint64
	applied uint64
	stabled uint64
	entries []pb.Entry
	pendingSnapshot *pb.Snapshot
}
```

- storage保存了自上次快照以来的所有已经stable状态的日志，持久化存储日志信息
- commited表示可提交的日志的上限，是大多数节点都保存了的日志的编号
- applied是已经应用到状态机的日志编号
- stabled为要持久化存储的日志上限
- entried[]是内存中保存的日志信息

当一个raft节点建立时使用`newLog`创建raft日志信息：

```go
func newLog(storage Storage) *RaftLog {
	// Your Code Here (2A).
	//  snapshot/first.....applied....committed....stabled.....last
	//  --------|------------------------------------------------|
	//                            log entries
	if storage == nil {
		log.Panic("storage must not be nil")
	}

	firstIndex, err := storage.FirstIndex()
	if err != nil {
		panic(err)
	}

	lastIndex, err := storage.LastIndex()
	if err != nil {
		panic(err)
	}
	hardstate, _, _ := storage.InitialState()

	entries, err := storage.Entries(firstIndex, lastIndex+1)
	if err != nil {
		panic(err)
	}
	r := &RaftLog{
		storage:         storage,
		committed:       hardstate.Commit,
		applied:         firstIndex - 1,
		stabled:         lastIndex,
		entries:         entries,
		pendingSnapshot: nil,
	}
	return r
}
```

newLog主要就从存储中加载日志信息到raft节点。

- `Raft.Prs[i].Match`是leader所认为的节点i已同步的日志编号
- `Raft.Prs[i].Next`是leader所认为的节点i下次要追加的日志编号
	* 之所以是leader认为是因为可能存在日志冲突的情况

leader想其他节点发送的日志复制请求中包含如下关键元素

```go
m := pb.Message{
	MsgType:              pb.MessageType_MsgAppend,
	To:                   to,
	From:                 r.id,
	Term:                 r.Term,
	LogTerm:              prevLogTerm,
	Index:                prevLogIndex,
	Entries:              entries,
	Commit:               leaderCommit,
}
```

- Term，当前leader的任期
- LogTerm，所追加的(第一个)日志所处的任期
- Index，所追加的(第一个)日志的编号
	* raft算法中如果任期号和日志编号都匹配则说明的同步的
- Entries，要追加的日志
- Commit，leader当前已经提交的日志编号

Raft中的日志复制是强一致性的，即如果follower和leader所保存的日志不一致，以leader为准，follower本地的日志将被leader覆盖。而判断日志是否一致就需要通过LogTerm和Index来判断。不一致的情况包括：

- follower日志缺失，则follower向leader回复自身当前最新日志的任期和编号，leader重新发送日志让follower同步
- follower日志冲突，则follower找到本地中第一个可能不冲突日志的任期和编号，向leader反馈

这里说可能不冲突是因为follower不知道之前所保存的日志是否和leader之前的日志冲突。所以为了加快follower和leader的同步速度，follower反馈时可以不以此退回一个日志，而是一次跳过所有冲突日志然后向leader发起同步。


### 执行流程

当客户端向leader发送指令，leader先是保存到本地日志然后使用`sendAppend`向其他节点广播：

```go
func (r *Raft) handlePropose(m pb.Message) {
	// leader本地添加日志
	es := make([]pb.Entry, len(m.Entries))
	lastIndex := r.RaftLog.LastIndex()
	for i := range m.Entries {
		es[i] = pb.Entry{
			EntryType: m.Entries[i].EntryType,
			Term:      r.Term,
			Index:     lastIndex + 1 + uint64(i),
			Data:      m.Entries[i].Data,
		}
	}
	// leader本地记录日志
	r.RaftLog.entries = append(r.RaftLog.entries, es...)
	// 更新leader自身的日志一致性信息
	r.Prs[r.id].Match = r.RaftLog.LastIndex()
	r.Prs[r.id].Next = r.Prs[r.id].Match + 1

	// 如果只有自己一个节点，commit直接更新
	if len(r.Prs) == 1 {
		r.RaftLog.committed = r.Prs[r.id].Match
		return
	}

	// 向其他节点广播，追加日志
	func() {
		for id := range r.Prs {
			if id != r.id {
				r.sendAppend(id)
			}
		}
	}()
}
```

向其他节点发送日志复制请求，会根据leader中记录的其他节点的日志性信息(`Raft.Prs[i].{Match, Next}`)向其他节点有序追加。并且为了保持强一致性，需要将leader所觉得的信息同日志信息一起发送给follower节点，follower节点根据这些信息可以判断是否与leader同步，不同步时则向leader反馈，发起同步。

值得注意的是`Raft.Entries`数组的索引和日志编号有微妙差别，因为日志可能累积了很多只加载一部分到内存`Raft.Entries`中。所以需要根据FirstIndex计算出编号i的日志在entries数组中的偏移。

```go
func (r *Raft) sendAppend(to uint64) bool {
	// Your Code Here (2A).
	var prevLogIndex uint64
	var prevLogTerm uint64
	var leaderCommit uint64
	var entries []*pb.Entry

	// leader所以为的，follower目前的最新日志。根据follower反馈，判断是否冲突
	prevLogIndex = r.Prs[to].Next - 1
	// says, 如果一个日志的任期号和日志id的正确的，那就是一致了的
	prevLogTerm, _  = r.RaftLog.Term(prevLogIndex)
	// 用于follower同步commit信息
	leaderCommit = r.RaftLog.committed

	// 更加日志ID, 计算日志在entries中的索引，生成日志entry，发送给follower
	firstLogIndex := r.RaftLog.FirstIndex()
	lastLogIndex  := r.RaftLog.LastIndex()
	var lo  = prevLogIndex + 1 - firstLogIndex
	var hi = lastLogIndex + 1 - firstLogIndex
	entries, _ = r.RaftLog.EntriesSlice(lo, hi)

	m := pb.Message{
		MsgType:              pb.MessageType_MsgAppend,
		To:                   to,
		From:                 r.id,
		Term:                 r.Term,
		LogTerm:              prevLogTerm,
		Index:                prevLogIndex,
		Entries:              entries,
		Commit:               leaderCommit,
	}
	r.send(m)
	return true
}
```

follower收到leader的日志复制请求信息，判断是否存在：日志冲突(LogTerm不一致)，日志缺失的不一致情况

- 如果一致(leader的日志是follower日志的子集的情况，$m.entries \subset r.RaftLog.entries$)，回复同意，节点本地更新commit信息
	* (`m.Index<=r.Index && r.LogTerm(min(m.Index,r.Index)==m.LogTerm)`)
- 如果不一致，回复拒绝，找出不一致的理由
	* follower任期更高(`m.Term < r.Term`)，拒绝
	* follower日志缺失(`m.LogTerm==r.LogTerm && r.Index<m.Index`)，返回节点最新的日志号，待leader发来同步
	* follower日志冲突(`m.LogTerm!=r.LogTerm && r.Index!=m.Index`)，找到第一个冲突的任期号


```go
func (r *Raft) handleAppendEntries(m pb.Message) {
	// Your Code Here (2A).
	lastIndex := r.RaftLog.LastIndex()


	if r.Term > m.Term {
		r.sendAppendResponse(m.From, true, lastIndex)
		return
	}
	// 否则leader任期更高，更新follower任期
	r.Term = m.Term
	
	// 如果follower已存在日志，则以leader为准(覆盖)
	if lastIndex >= m.Index {
		lastIndex = m.Index
	}

	logTerm, _ := r.RaftLog.Term(m.Index)
	// 只有follower已存在日志或follower日志与leader同步的情况
	// 才有可能同意日志复制请求
	if lastIndex == m.Index && logTerm == m.LogTerm {

		var length = len(m.Entries)
		if len(r.RaftLog.entries) != 0 {
			var existed = 0
			// |  lastIndex    |
			// |  m.index | m.Entries	|
			for _, entry := range m.Entries {
				term, _ := r.RaftLog.Term(entry.Index)
				// 如果已存在的日志与leader发送的相同
				// 可以更新可以跳过覆盖，然后更新commit信息
				if entry.Term != term {
					break
				}
				existed++
			}

			lastIndex = m.Index + uint64(existed)
			firstIndex := r.RaftLog.FirstIndex()
			m.Entries = m.Entries[existed:]
			// leader日志都已存在，且一致，跳过删除，原stable至少没发现冲突
			if len(m.Entries) != 0 {
				r.RaftLog.entries = r.RaftLog.entries[:lastIndex+1-firstIndex]
				// 1. Term的情况
				if existed != 0{
					r.RaftLog.stabled = m.Index
				} else {
					// 2. 失效的日志: stable过新
					if m.Index <= r.RaftLog.stabled {
						r.RaftLog.stabled = m.Index
					}

				}
			}
		}
		for _, entry := range m.Entries {
			r.RaftLog.entries = append(r.RaftLog.entries, pb.Entry{
				EntryType:            entry.EntryType,
				Term:                 entry.Term,
				Index:                lastIndex+1,
				Data:                 entry.Data,
			})
			lastIndex++
		}

		lastIndex = r.RaftLog.LastIndex()
		if m.Commit > r.RaftLog.committed {
			// commit应当是不超过leader commmit的最新编号
			committed := min(m.Commit, m.Index+uint64(length))
			r.RaftLog.committed = min(committed, r.RaftLog.LastIndex())
		}

		r.sendAppendResponse(m.From, false, lastIndex)
		return
	}

	hintIndex := min(m.Index, r.RaftLog.LastIndex())
	hintIndex = r.RaftLog.findConflictByTerm(hintIndex, m.LogTerm)

	r.sendAppendResponse(m.From, true, hintIndex)
	return
}
```

在tinykv中，leader日志覆盖时不是直接覆盖，如果要最佳的日志follower中都已存在了则可以跳过，即使follower后续还有日志。这么做可以减少io开销。

follower处理日志追加请求，如果发生了日志冲突，通过`findConflictByTerm()`找到最新可能不冲突的日志，反馈给leader尝试同步。leader也根据follower的反馈找最新可能不冲突的日志向follower同步。这样加快了同步的过程，而且最终总会从早前某个位置已经同步的日志开始同步。

leader收到follower的反馈，如果没有日志冲突则leader更新该follower节点的同步信息，然后判断是否需要更新commit。


```go
if m.Reject == false {
	r.Prs[m.From].Match = m.Index
	// r.Prs[m.From].Match = r.Prs[m.From].Match + 1
	r.Prs[m.From].Next = m.Index + 1

	// 更新日志commit信息
	r.updateCommited(m)
}
```

更新commit是由大多数节点的同步情况决定的，如果大多数节点都同步到了日志i则commit更新为i。这里为了统计大多数节点的最新同步情况，可以使用排序后求中数的算法进行优化。

```go
func (r *Raft) updateCommited(m pb.Message) {
	// 单纯遍历，然后计数一次只能更新一个commit
	// 排序然后取中位数，就直接得到了大多数节点的commit
	match := make(uint64Slice, len(r.Prs))
	i := 0
	for _, prs := range r.Prs {
		match[i] = prs.Match
		i++
	}
	sort.Sort(match)
	// -1是为了保证真正是大多数，因为可能存在偶数的情况
	Match := match[(len(r.Prs)-1)/2]
	matchTerm, _ := r.RaftLog.Term(Match)
	if Match > r.RaftLog.committed && matchTerm == r.Term {
		r.RaftLog.committed = Match
		// 广播节点也更新commit信息
		for peer := range r.Prs {
			if peer != r.id {
				r.sendAppend(peer)
			}
		}
	}
}
```

如果存在冲突leader要根据follower的反馈`findConflictByTerm`找到最新可能不冲突的日志编号，从这个日志开始发起同步。如果仍不同步则重复此过程。

```go
if m.Reject == true {
	nextProbeIdx := r.RaftLog.findConflictByTerm(m.Index, m.LogTerm)

	r.Prs[m.From].Next = max(min(m.Index, nextProbeIdx+1), 1)
	r.sendAppend(m.From)
}
```

