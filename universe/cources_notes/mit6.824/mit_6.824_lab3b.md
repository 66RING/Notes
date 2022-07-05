---
title: MIT6.824 lab 3B 记录与bug总结
author: 66RING
date: 2022-05-17
tags: 
- MIT
- distributed
mathjax: true
---

# MIT6.824 lab 3B bug总结记录

> 到目前lab3为止，可以导致状态变更的，有潜在是不一致风险的事件有：状态变更，日志追加，日志恢复
> 
> 这些事件在我看来是类似操作系统中"中断"的存在，但我目前还无法用我的语言来抽象概括，大概是要防止mock的序列被破坏吧，总之需要多加小心

bug总结概括：引入"Snapshot中断"后，对整个raft集群状态的控制大失败。TODO：抽象

## lab2遗留bug1

在lab2中，因为没有因为kvserver层，我对`Snapshot`, `InstallSnapshot`和`CondInstallSnapshot`其实是不理解的。我只知道单元测试会在`InstallSnapshot`通知`applyCh`后调用`CondInstallSnapshot`然后执行`Snapshot`进行日志压缩。因为我的视野只看到了raft层，所以我就认为如果`InstallSnapshot`的判断通过了，通过`applyCh`了那`CondInstallSnapshot`有什么用？于是我在`InstallSnapshot`通过后就进行了状态变更，然后`CondInstallSnapshot`恒返回true

```go
func(rf *Raft)InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply){
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if !invalid(args) {
		return
	}

	// 直接日志恢复
	if args.LastIncludedIndex <= rf.LastIndex() && args.LastIncludedIndex >= rf.LastSnap().Index &&
		rf.LogTerm(args.LastIncludedIndex) == args.LastIncludedTerm {
		idxTrim := rf.offset(args.LastIncludedIndex)
		rf.dropLog(idxTrim)
	} else {
		rf.log = make([]Entry, 1)
		rf.log[0].Term, rf.log[0].Index = args.LastIncludedTerm, args.LastIncludedIndex
	}

	rf.lastApplied, rf.commitIndex = args.LastIncludedIndex, args.LastIncludedIndex
	rf.becomeFollower(args.Term, args.LeaderId)

	rf.persist()
	rf.persister.SaveStateAndSnapshot(rf.persister.ReadRaftState(), args.Data)

	go func() {
		rf.applyCh <- ApplyMsg{
		}
	}()
}
```

实际应该是`InstallSnapshot`通知`kvserver`层日志恢复后再日志恢复安装kv数据库的状态等。

```
InstallSnapshot -> kvserver
                   ...
                   kvserver: <- ch
				   ...
                   kvserver: if CondInstallSnapshot == true {RecoverSnapshot()}
```

而我错误的实现是`InstallSnapshot`时已经将raft部分的内容安装了，而kvserver部分会稍后再安装。那么在这两个事件的空隙中就可能出错。考虑如下情况：

1. S1(leader)刚刚日志压缩，然后给S2发送`InstallSnapshot`
2. S2收到`InstallSnapshot`更新节点的log和Term，**此时他的日志和任期和S1一样新，将有资格成为leader**
3. S2发起竞选，成为leader，但是S2的kvserver层并没更新且`applyCh`中"积压"了一些ApplyMsg
4. 这时S2应用"积压"的ApplyMsg并返回给client，然后导致不一致
5. 最后`RecoverSnapshot`才姗姗来迟，沦为"最终一致性"


## lab2遗留bug2

`Snapshot()`函数里加锁导致死锁，我在2D时没能解决，于是使用一个goroutine包装一笔带过了，结果后面出现了大问题。

首先是我的实现为什么会死锁：

因为我`rf.applyer`里，锁保护了`ch <-`，而这个ch会导致接收端触发Snapshot，如在kvserver的`applyer()`中接收到apply通知。这样apply通知就可能触发Snapshot(到达阈值)

```go
func (rf *Raft)applyer() {
	rf.mu.Lock()
	(1)
	for() {
		ch <- 
	}
	updateLastApplied()
	rf.mu.Unlock()
}

func (kv *KVserver)applyer() {
	kv.mu.Lock()
	<- ch
	(2)
	Snapshot()
	// Snapshot
	// 	rf.mu.Lock()
	// 	rf.mu.Unlock()
	kv.mu.Unlock()
}
```

这样在(2)`ch<-`后,`Snapshot()`执行lock前就可能死锁：在(2)时(1)中`ch<-`继续执行，没有接收方就死锁了。

解决办法是`rf.applyer()`中的`ch<-`不加锁。如果这样的话`updateLastApplied()`要在`ch<-`解锁前，因为解锁后其他地方可能用到`lastApplied`。如中途插入执行`InstallSnapshot`，更新了`lastApplied`。如果`updateLastApplied()`后执行，`InstallSnapshot()`更新的`lastApplied`就白费了，引入额外开销。

其次，开goroutine做`Snapshot`的问题，因为`Snapshot`这个协程不一定能第一时间拿到锁，也就是说日志压缩触发了但没开始压缩，所以在`Snapshot`抢到锁前，后续的日志将一致触发日志压缩(因为日志已经达到阈值)。


## lastApplied更新问题导致应用遗漏

下面是我的错误的实现, 错误有几点：

1. 错在直接访问`log`原地
	- 不是说不行，只是在`Unlock`后原子性打破了，破在仍是`idxLastApplied`和`idxCommitIndex`的区间
2. 错在`rf.lastApplied++`

上述两点错误的**根本原因**是，引入日志压缩和日志恢复后，这整个`for`循环不再是原子的了，for并不知情log等已经更新，即中途可能执行`InstallSnapshot`导致`lastApplied`和log更新。

`lastApplied`更新将可能导致**下一轮for的apply**出现日志遗漏，没能apply。`log`更新将可能导致数组溢出，直接panic。

```go
for _, ent := range rf.log[idxLastApplied+1:idxCommitIndex+1] {
	applyMsg := ApplyMsg{
		CommandValid:  true,
		Command:       ent.Command,
		CommandIndex:  ent.Index,
	}
	rf.lastApplied++;
	rf.mu.Unlock()
	rf.applyCh <- applyMsg
	rf.mu.Lock()
}
```

有两种修改方案：

1. 在apply的`for`循环中判断是否需要apply，否则break进入等待
2. 创建一份待apply的日志的快照，即`copy`，然后根据该快照apply，这种方式可能会通知重复的情况，**依赖kvserver层的过滤**

最后我采用了第二种：

```go
entries := make([]Entry, idxCommitIndex-idxLastApplied)
copy(entries, rf.log[idxLastApplied+1:idxCommitIndex+1])
rf.lastApplied = rf.commitIndex
rf.mu.Unlock()
for _, ent := range entries {
	applyMsg := ApplyMsg{
		CommandValid:  true,
		Command:       ent.Command,
		CommandIndex:  ent.Index,
	}
	rf.applyCh <- applyMsg
}
rf.mu.Lock()
```


## 引入InstallSnapshot后投票状态丢失

我的错误实现中，每次如果`InstallSnapshot()`合法，则接收方将`becomeFollower()`，而`becomeFollower()`会让`votedFor=-1`。这样**一个任期只能选举出一个leader**就被破坏了。这些`votedFor`丢失的节点可以投票给新leader，从而导致多leader。

错误代码形如：

```go
func(rf *Raft)InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply){
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if invalid(args) {
		return
	}

	rf.becomeFollower(args.Term, args.LeaderId)

	go func() {
		rf.applyCh <- ApplyMsg{
			CommandValid:  false,
			SnapshotValid: true,
			Snapshot:      args.Data,
			SnapshotTerm:  args.LastIncludedTerm,
			SnapshotIndex: args.LastIncludedIndex,
		}
	}()
}
```

修正：只在发送方的Term大于当前Term时`becomeFollower`，等于是不可以的，因为可能是当前leader(相同任期)发的

```go
func(rf *Raft)InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply){
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if invalid(args) {
		return
	}

	if rf.currentTerm < args.Term {
		rf.becomeFollower(args.Term, args.LeaderId)
	}

	go func() {
		rf.applyCh <- ApplyMsg{
			CommandValid:  false,
			SnapshotValid: true,
			Snapshot:      args.Data,
			SnapshotTerm:  args.LastIncludedTerm,
			SnapshotIndex: args.LastIncludedIndex,
		}
	}()
}
```


## 重复请求的过滤函数问题

我早先判断重复的函数如下，是居然是判断最近的id和待添加的id**相等**: `resp.LastSeqNum == seqNum`。因为我只考虑了"一个client一次只会发送一个请求"。也就是我一位重复的请求的id一定是上一次的请求id。

```go
func (kv *KVServer) isDuplicate(clientId int64, seqNum int64) bool {
	resp, exist := kv.responseTracker[clientId]
	if(exist && resp.LastSeqNum == seqNum) {
		return true
	}
	return false
}
```

但是当引入日志压缩机制后，就需要考虑两个事件的乱序：日志恢复和日志追加。

1. 日志恢复后可能`LastSeqNum`记录被提高了：`LastSeqNum > seqNum`因为是不等于，所以可以通过过滤导致重复
2. 又要考虑Get请求无论如何都能通过导致的错误更新，如Get导致lastSeqNum回退

因为我的判断是`!isDuplicate || OpGet`，那么`OpGet`通过后将导致`LastSeqNum`回退，然后后面的重复日志就被应用。

**错误代码形如**

```go
if !kv.isDuplicate(op.ClientId,op.SequenceNum) || op.Type == OpGET {
	res.Value, res.Err = kv.execute(op)
	// update new seqNum
	kv.responseTracker[op.ClientId] = Response{
		Value:      res.Value,
		Err:        res.Err,
		LastSeqNum: op.SequenceNum,
	}
}
```

改：因为get操作必须执行，所以OpGet不记录响应

```go
if !kv.isDuplicate(op.ClientId,op.SequenceNum) {
	res.Value, res.Err = kv.execute(op)
	// update new seqNum
	kv.responseTracker[op.ClientId] = Response{
		Value:      res.Value,
		Err:        res.Err,
		LastSeqNum: op.SequenceNum,
	}
}else if op.Type == OpGET {
	res.Value, res.Err = kv.execute(op)
}
```

## lab2遗留bug3

我对日志追加的处理理解有误，我在成功追加时，返回了该follower的最后一个日志的Index。而leader会根据这个返回Index更新matchIndex，而实际应该match的是leader发送的最后一个Entries的Index。即

```go
func (rf *Raft)HandleAppendEntries(args *MsgArgs, reply *MsgReply) {
	if success == true {
		reply.Term = rf.currentTerm
		reply.LogTerm = rf.LastTerm()
		reply.LogIndex = rf.LastIndex()
		reply.Success = true
		return
	}
}
```

然后在leader端，leader就会以为大部分的follower都备份了，然后commit。然而其实并没有一致，提交将导致不安全状态。

```go
if reply.Success == true {
	rf.matchIndex[server] = min(reply.LogIndex, rf.LastIndex())
	rf.nextIndex[server] = rf.matchIndex[server] + 1
	rf.updateCommit()
}
```

修改：把`reply`的成员名改为以`Conflict`前缀，提醒我自己该函数的逻辑。然后leader端更新是根据发送的日志的Index进行更新才对，"leader询问的和leader收到的回复应该是针对同一个对象"。


```go
func (rf *Raft)HandleAppendEntries(args *MsgArgs, reply *MsgReply) {
	if success == true {
		reply.Term = rf.currentTerm
		// reply.ConflictLogTerm = rf.LastTerm()
		// reply.ConflictLogIndex = rf.LastIndex()
		reply.Success = true
		return
	}
}
```

```go
if reply.Success == true {
	if len(args.Entries) != 0 {
		prevIndex := args.Entries[len(args.Entries)-1].Index
		rf.matchIndex[server] = min(prevIndex, rf.LastIndex())
		rf.nextIndex[server] = rf.matchIndex[server] + 1
		rf.updateCommit()
	} else {
		rf.NoticeApply()
	}
}
```





