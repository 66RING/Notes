---
title: MIT6.824 lab 2 记录与bug总结
author: 66RING
date: 2022-05-26
tags: 
- MIT
- distributed
mathjax: true
---

# MIT6.824 lab 2 记录与bug总结

两个主要问题：乱序RPC和"Figure 8问题"(小论文Figure 8, 大论文Figure 3.7)

> 两阶段写中间有gap，你以为的原子也许不那么原子不那么livenss


## 论文Figure 8中描述的问题

Figure8问题：leader的日志被另一个leader的日志覆盖

根源: **两阶段写, 而中间gap时没"互斥"**

举个例子：

```
(1) S1当选leader Term=2，并追加日志到2
S1: 1 2
S2: 1 2
S3: 1
S4: 1
S5: 1

(2) S1没能收到2的响应，就crash了。S5当选 Term=3，处理一个请求添加日志
S1: 1 2
S2: 1 2
S3: 1
S4: 1
S5: 1 3

(3) S5当选后自在自己的日志里做了记录就crash了
      不过它的心跳信号让(2, 3, 4)的Term都成了3
    这时S1, S2可以成为Leader，因为他们的日志更新
	假设S1当选 Term 4，然后日志追加到S3
S1: 1 2
S2: 1 2
S3: 1 2
S4: 1
S5: 1 3

**
** 此时
**
1. leader(S1)任期4, lastLog的Term是2
2. S5 crash, 任期3，并不知道自己已经不是leader了

如果这种情况下leader可以commit，即leader当前的任期和大多数节点日志的任期不一致的情况下也能commit

(4)
S1 commit, 已commit的日志所属任期为: 1 2

(5) 这时leader S1 crash, S5恢复，S5通过投票请求等自己的任期也变为了4
    并且最后S5很幸运的当选(毕竟任期相同的情况下，S5的lastLog的Term更高)
    **最后S5的日志将可以覆盖之前已经提交的日志**
S1: 1 2
S2: 1 3
S3: 1 3
S4: 1
S5: 1 3 5

"历史被改变了"导致不安全
```

> To eliminate problems like the one in Figure 8, Raft
> never commits log entries from previous terms by counting 
> replicas. Only log entries from the leader’s current
> term are committed by counting replicas;

Figure8问题的解决方案是说: 当大多数节点日志复制到达一个Index，**且这个Index的日志所属的任期和leader当前任期相同时** ，Leader才能根据这点更新commitIndex。然后才能从lastApplied的日志应用到commitIndex的日志。

> 费曼说，我不能实现的东西我无法理解，所以我就对这个情况做一个肤浅的抽象:
> 
> 上述不安全问题的根本原因是没能保证"互斥"，或者说 **intersection**，没有互斥一致就没有保证。什么叫互斥? Raft处处都在强调的大多数节点, majority就是互斥。谁和谁的互斥? 我认为应该是同一个任期的所有事件的互斥，不过在这个例子中可以说是vote和commit的互斥(commite和修改日志互斥)。都是认准majority怎么就不互斥了？所在的任期不同，不可以跨次元互斥哦/必须同台(Term)竞技。
> 
> 1. 长日志节点
> 2. 中日志节点
> 3. 短的stable的日志节点
> 
> 长日志commit条件："Only log entries from the leader’s current term are committed by counting replicas"。为了让短日志节点能够反抗中日志节点(`lastLogTerm`大)。否则短日志节点可被中日志和长日志节点修改，导致"race"。
> 
> damn 好菜 我什么时候才能用符号做抽象啊

Q: 如果新leader当选后一直没有新请求，还没commit的日志是不是就是永远得到不commit了?

A: yes，当选leader后应该广播一个`[{nil}]`的什么都不敢的日志的，而如果6.824中这么干在一个测试就无法通过，我觉得是6.824设计的问题。所以leader当选后应该提交一个空日志。



## 乱序RPC

> Receiver implementation:
> 1. Reply false if term < currentTerm (§5.1)
> 2. Reply false if log doesn’t contain an entry at prevLogIndex
> 	whose term matches prevLogTerm (§5.3)
> 3. If an existing entry conflicts with a new one (same index
> 	but different terms), delete the existing entry and all that
> 	follow it (§5.3)
> 4. Append any new entries not already in the log
> 5. If leaderCommit > commitIndex, set commitIndex =
> 	min(leaderCommit, index of last new entry)

论文中，似乎没说"如果append的日志已存在且不冲突"时该怎么处理。应该很容易想到，无脑截断，然后追加呗。如果这样将导出不安全情况的出现，正确的做法应该是**不冲突就不删除**，因为可能会出现 **乱序RPC** 的情况：

> 我刚开始看到"不冲突就不删除"时还以为是降低开销，结果不是

```
[2 <- 4]: Entries 表示节点2收到的日志
[2 <- 4] 后两行表示节点2的日志状态
RPC1
2022/02/09 11:38:23 [2 <- 4]: Entries: [{3 2 319} {4 2 9827}], pIdx 2, pTerm 1
2022/02/09 11:38:23 [2 <- 4] T2, 原始 logs: [{0 0 0} {1 1 1830} {2 1 7432}]
2022/02/09 11:38:23 [2 <- 4] T2, 修改 logs: [{0 0 0} {1 1 1830} {2 1 7432} {3 2 319} {4 2 9827}]

RPC 2
2022/02/09 11:38:23 [2 <- 4]: Entries: [{3 2 319}], pIdx 2, pTerm 1
2022/02/09 11:38:23 [2 <- 4] T2, 原始 logs: [{0 0 0} {1 1 1830} {2 1 7432} {3 2 319} {4 2 9827}]
2022/02/09 11:38:23 [2 <- 4] T2, 修改 logs: [{0 0 0} {1 1 1830} {2 1 7432} {3 2 319}]
```

考虑如下情况，S4发出个RPC的顺序是RPC2, RPC1，而S2收到RPC的顺序是RPC1，RPC2。无脑截断的话，乱序RPC将导致原本已经已经追加的日志丢失了。

1. RPC1的返回然leader满足commit条件，然后提交，状态机有`1| 2|`这两个日志
```
S1: 1| 2|
S2: 1| 2|
S3: 1| 2|
S4: 1|
S5: 1|
```
2. 乱序的RPC2悄悄地修改了follower(假设`S3`)的日志:
```
idx:0  1
S1: 1| 2|
S2: 1| 2|
S3: 1|
S4: 1|
S5: 1|
```
4. 发生了选举超时`S3`得到4 5的投票成为leader，然后收到新指令。这样figure 8的情况就发生了，出现了不一致的情况，no safety
```
idx:0  1
S1: 1| 2|
S2: 1| 2|
S3: 1| 3|
S4: 1| 3|
S5: 1| 3|
```


## 处理过期RPC

调用RPC一定不能加锁吧，则解锁这段期间状态可能改变。RPC结束后要处理返回吧，那需要注意**RPC返回后和调用RPC时的statement是否符合预期**。

```go
if rf.state != Leader || args.Term != rf.currentTerm {
	return
}
```


## 锁用法错误，原子性破坏1

考虑下面的模型，就是一个函数，准备要发送的日志，发送，处理回复。

```go
for i := range rf.peers {
	go rf.sendAppend(i)
}

func (rf *Raft)sendAppend(server int) {
	lock()
	1. 根据matchIndex和nextIndex准备Entries
	unlock()
	2. 调用日志追加RPC，因为rpc等待，所以不加锁
	lock()
	3. 根据返回调整matchIndex和更新commit等
	4. 如果reply.Term > currentTerm的话还要改变状态. (All Servsers Rules)
	unlock()
}
```

问题就出在**锁** 和 **第`4`点**。因为是用goroutine发送，所以在获取锁时我的`sendAppend`会进入等待队列等待，如果这时触发了`4`发生了状态转换，后续的`sendAppend`都会继续。而且我的状态转换会让当前节点的`currentTerm = reply.Term`，因为这也是rule说的。一旦`currentTerm = reply.Term`就麻烦了，这意味着原先因为Term追加不上现在可以追加了。而且发出追加的节点不再是leader了，但是它仍然接收返回，检测commit，从而导致commit错误。

所以我在第`1`点之前添加了判断，如果节点不再是`leader`就返回。

```go
if rf.state != Leader {
	rf.mu.Unlock()
	return
}
```


## 锁用法错误，原子性破坏2

> 又是"等待队列"无法撤回问题
>
> 又一个6.824测试设计的问题，一个不重试的问题

ticker有一个超时循环，通过`electionResetCh`来重置计时器

```go
go func() {
	for rf.killed() == false {
		select {
		case <- rf.electionResetCh:
		case <- time.After(RandElectionTime()):
			rf.mu.Lock()
			rf.becomeCandidate()
			rf.mu.Unlock()
		}
	}
}()
```

考虑如下情况：

1. S2向S1发送投票请求，S1同意
2. 但是S1同意前的瞬间选举超时，进入到下面的case等待获取锁，此时S1已经在超时的case中
```go
case <- time.After(RandElectionTime()):
	rf.mu.Lock()
	rf.becomeCandidate()
	rf.mu.Unlock()
}
```
3. 然后回到`RequestVote`，会重置S1的计时器，然后S1回复赞成
4. **此时S1虽然重置了计时器，但还是进入了超时**
5. 又因为在`RequestVote`中Term提升了，超时后becomeCandidat Term进一步提升

比如在上述情况中，S2得到了S1同意当选了leader，但是S1的Term更大，然后请求发送给leader S2。但是因为S1的Term更大，很快就成为leader然后覆盖上一个请求。虽然不会就让不安全状态，但是在MIT6.824的`TestFailNoAgree2B`测试中使用`cfg.one(101, servers, false)`，其中`retry=false`，那么如果第一次`Start`丢失了那就是失败了。

所以可以考虑锁移出case

```go
for {
	time.Sleep(rf.heartBeat)
	rf.mu.Lock
	if time.Now().After(rf.electionTime) {
		rf.becomeCandidate()
	}
	rf.mu.Unlock
}
```

> `(t time.Time).After(u time.Time)`会检查实例`t`是否经过了u的时间间隔
> 
> 这样以来偶尔(`headBeat`)会检查以下。避免上述问题


## 如何理解matchIndex

matchIndex就是leader认为的，其他节点已经保存到的日志index。所以在一个leader当选时，所有matchIndex应该置0，防止leader错误的以为可以commit。


## VoteRequest重置计时器问题

当一个节点收到一个`args.Term > rf.currentTerm`的投票请求时，不应该重置该节点自身的超时器。因为这个节点的日志可能更新，那么刷新了计时器就相当于抑制了"真正的leader"的产生。


## 2D API

`Persister`提供了一些方法包将数据保存到`persister`成员中，注意使用Persister API


```go
func (ps *Persister) SaveRaftState(state []byte) {
	ps.mu.Lock()
	defer ps.mu.Unlock()
	ps.raftstate = clone(state)
}

func (ps *Persister) ReadRaftState() []byte {
	...
}

func (ps *Persister) SaveStateAndSnapshot(state []byte, snapshot []byte) {
	ps.mu.Lock()
	defer ps.mu.Unlock()
	ps.raftstate = clone(state)
	ps.snapshot = clone(snapshot)
}
func (ps *Persister) ReadSnapshot() []byte {
	...
}
```
