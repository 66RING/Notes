---
title: MIT6.824 lab 3A 记录与bug总结
author: 66RING
date: 2022-05-13
tags: 
- MIT
- distributed
mathjax: true
---

# MIT6.824 lab 3A 记录与bug总结

debug真的很搞心态，不过对理解很有帮助(我觉得，当然看别经验贴可能效率会更高)。anyway如果一次都没跌倒过，我也许永远都注意不到这些bug呢

所以这里 **严重剧透警告**， **严重剧透警告** ， **严重剧透警告** 

欢迎批评指正，欢迎讨论补充，欢迎分享自己遇到的corner case


## 3A

### fuck the lab

1. 出问题时不要害怕再次重头阅读你写的代码(log看半天看不出来)
2. 理解测试很有必要
	- `p1, p2 := make_partition()`制造网络分区，然后把当前leader放到minority(p2)中
3. 乱序无处不在

### bug记录

1. 小心一切脱离了raft mock序列的问题
2. 小心一切二次加工raft mock序列的问题
3. TODO: summary

#### 过滤重复请求

> 1. 在kv的applyer中过滤
> 2. 优化：在接收client请求时可以可以过滤，用于加速**但是比较tricky**

首先说明一下我的实现的大概流程：

```go
func CommandHandler() {
	kv.rf.Start()
	<- kv.watchCh
}

go func applyer() {
	msg <- kv.rf.applyCh
	execute(msg)
}
```

- `CommandHandler`负责接收客户端的请求，然后通过`Start()`交给raft
- `applyer`接收到raft层的commit后应用到状态机

考虑client重复请求的问题，以下是我的经历：

1. 仅在`CommandHandler`中，`Start()`调用前，根据client携带的seqNum判断重复请求，然后过滤
	- 为什么我要先在`CommandHandler`中判？因为我觉得，如果在`applyer`中判，那么raft日志中就会有很多重复的日志，因为没经过过滤直接`Start()`
2. 仅在`applyer`执行(`execute`)前判断
	- 为什么这时我把`CommandHandler`中的判断删了？因为我觉得两头判引入复杂性
3. 在`CommandHandler`和`applyer`中都判断重复
	- 减少raft日志压缩的频率，并过滤重复请求

以下是结论：

1. 首先第一个方案是不对的，考虑等待commit超时的情况
	- 首先为了防止网络失败的情况，等待commit失败了就应该将记录的"lastApplied"信息回滚
	- 这个回滚就出现大问题了，**因为command可能已经在log里了，只是还没执行到** ，这样也回滚后相同的命令将可以再次进入raft log导致重复请求
2. 第二个方案呢就会在raft log中记录大量重复情况，在apply时才过滤
	- 问题就raft log大量重复，触发日志压缩频率高，
3. 在`CommandHandler`和`applyer`中都判断重复
	- `CommandHandler`中的过滤减少raft log中的重复
	- `applyer`中个过滤方式重复应用到状态机


> 论文中类似"滑动窗口"的解决方案

- 为什么必须使用滑动窗口? 主要是考虑一个client可以同时发送多个请求的情况
	* 首先mit6.824中是不用担心的，因为已经说明一个client一次只会发送一个请求
	* 发送多个请求时需要考虑乱序和rpc fail，如果不把历史回复缓存(滑动窗口)，那么在rpc fail时和乱序时，seqNum递增，低seqNum就会被高seqNum覆盖，从而导致低seqNum的请求永远得不到执行


#### 正确使用监听用的channel

> 使用map封装chan，让写者可以监听chan的关闭

kvserver层需要一种机制来监听raft层的apply事件，然后应用到kv状态机。ok基于这个目的很容易想到在go中可以使用channel来方便的实现。不过这可能存在一个问题：**写者如何判断chan是否关闭**。

一般来说规定chan写者是负责关闭的，但是因为我们的kvserver是要超时重试的，所以在channel的另一端会超时然后关闭channel，这就导致已关闭的channel写的问题。

所以可以使用一个`map`来封装chan，关闭chan的时候从map删除，写时判断map中是否存在。

```go
// bug
watchCh[idx] <- res

// fix
if ch, ok := watchCh[idx]; ok {
	ch <- res
}
```


#### 非线性一致性bug: 用log的Index做索引存在的问题

> 怎么索引watchCh: 使用uuid

我的bug就出在我用了`Start()`返回的index做索引：`index, _ , _ := kv.rv.Start()`。这会出现什么问题？因index在节点间是可能出现冲突的，最终会和leader一致。所以设想两个leader, `L1`, `L2`，我们的节点`cur`发给`L1`，但是发生了网络分区`L1`其实已经不是leader了。而如果此时`L2`的index和`L1`的一致，最后`L1`一定会被覆盖，并且**错误的以为该index是客户端等待index**从而导致不一致的情况发生。

```
// (1)
// 开始 1是leader
1,2,3,4,5: log1 | log2 |

// (2)
// 分区 1不知道自己已经不是leader了
1,5	 : log1 | log2 |
2,3,4: log1 | log2 |

// (3)
// client1 向1发，不会得到大多数节点同意，重试
// 2当选
// client2 向2发，得到大多数节点同意，广播，节点复制
1,5	 : log1 | log2 | log_new1(index3)
2,3,4: log1 | log2 | log_new2(index3)

// (4)
// 网络恢复，节点1被节点2日志覆盖
// 但是该index3不是client1真正期望的
1,5	 : log1 | log2 | log_new2(index3)
2,3,4: log1 | log2 | log_new2(index3)
```

TODO 全局uuid办法

我其实也没有解决这个问题，使用一个大随机数做uuid然后索引。


#### 注意Decode一个`type interface`的问题

> golang不熟悉：type interface内存模型，具体含义

因为我的实现中，kv数据库是一个interface，以方便扩展:

```go
type KvDb interface {
	Get(key string) (string, Err)
	Put(key string, value string) Err
	Append(key string, value string) Err
	DB() *map[string]string
}

type KVServer struct {
	database KvDb
}
```

这时decode就会出现**golang菜鸟问题**，如我是这么decode的：

```go
var database KvDb
if err := d.Decode(&database); err == nil {
	kv.database = database
}
```

类型匹配了，也没有报错。不过这么是解码不出来任何东西的。

改成下面这样就可以了:

```go
var database BasicDB
if err := d.Decode(&database); err == nil {
	kv.database = &database
}
```

**TODO 学习golang的`type interface{}`，学习其内存模型**


#### 死锁

> kv层锁后不要再尝试对raft层加锁
>
> 本质原因是并发导致的AB,BA型锁

```go
func CommandHandler() {
	kv.mu.Lock()
		(1)
		kv.rf.mu.Lock()
		kv.rf.mu.Unock()
	kv.mu.Unock()
	Start()
	...
	<-watchCh
	...
	(2)
	kv.mu.Lock()
	kv.mu.Unock()
}

kv.applyer() {
	kv.mu.Lock()
	(3)
	<- applyCh
	kv.mu.Unock()
}

rf.applyer() {
	kv.rf.mu.Lock()
	(4)
	applyCh <-
	kv.rf.mu.Unock()
}
```

死锁是这样子的：

```
 (1)      (4) 			  (3) 			 (1)
 --->  rf.lock  --->  kv.applyCh  ---> kv.lock
```

所以我的答案是: 别把rf锁嵌套在kv锁内。


#### 测试超时问题

##### GET的处理

> get不存在时直接返回空字符串和OK

我在get不存在的key的时候的处理有大问题，导致client一直重试，然后测试超时。

因为实验文档说在get不存在的key时返回空字符串`""`，我是返回了，但是还返回了一个NoKey错误。然后client端`for reply.Err != Ok{重试}`，这就导致了超时


##### watchCh阻塞问题

> watchCh待缓冲，因为读者可能跑路了

`watchCh`要待1个缓冲。因为读者在监听时是不可能加锁的，所以就就可能不再监听后写者立刻写导致阻塞。即`(1)`后立刻调度到`(2)`

```go
{
	select {
	case res := <- ch:
		value, err = res.Value, res.Err
	case <- time.After(ExecuteTimeout):
		value, err = "", ErrTimeout
	}
	(1)
	kv.mu.Lock()
	delete(kv.watcher, index)
	kv.mu.Unlock()
}

applyer() {
	kv.mu.Lock()
	if ch, ok := kv.watcher[op.WatchChId]; ok {
		(2)
		ch <- res
	}
	kv.mu.Unlock()
}
```

`watchCh`要待1个缓冲。因为读者在监听时是不可能加锁的，所以就就可能不再监听后写者立刻写导致阻塞。即`(1)`后立刻调度到`(2)`

```go
{
	select {
	case res := <- ch:
		value, err = res.Value, res.Err
	case <- time.After(ExecuteTimeout):
		value, err = "", ErrTimeout
	}
	(1)
	kv.mu.Lock()
	delete(kv.watcher, index)
	kv.mu.Unlock()
}

applyer() {
	kv.mu.Lock()
	if ch, ok := kv.watcher[op.WatchChId]; ok {
		(2)
		ch <- res
	}
	kv.mu.Unlock()
}
```


#### Test: completion after heal (3A)

> RPC fail 和 WrongLeader都要换一个leader重试

RPC fail和WrongLeader都换leader，因为RPC fail说明该节点大概率也挂了，而且该"少数派的leader"并不会知道自己已经不是leader(因为仍能收到心跳)，如果不换leader重新尝试，并且网络恢复不及时那么该请求会一种超时。


#### 为什么必须保存上一次的响应

> 基本思想类似TCP滑动窗口，buffer的本质
>
> 防乱序，有backup

`responseTracker`不单单是为了"加速"，还防止了RPC fail。因为要过滤重复请求，client端要携带一些标识的信息，server根据这些信息判断重复。RPC fail时就可能出现server觉得这个client的已经接收到结果了发了个重复的请求，而实际的client是并没有收到结果所以重发请求。所以就会server不断以为是重复请求拒绝，client不断收不到结果。


#### TestSpeed

关闭数据竞争检测功能


## 3B

[MIT6.824 lab 3B 记录与bug总结](https://github.com/66RING/Notes/tree/master/universe/cources_notes/mit6.824/mit_6.824_lab3b.md)


## Gross

- `CommandHandler`处理client请求的入口函数，等待apply后返回结果
- `watchCh`, applyer用于通知`CommandHandler`apply情况的channel
- `tracker`, `responseTracker`, `session`, `sessonTracker`
	* 都是论文中说的用于记录"上一次response"的结构
- `seqNum`, 客户端发送请求时携带的用于标识次序的id，单调递增



