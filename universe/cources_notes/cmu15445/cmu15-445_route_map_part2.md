---
title: CMU15-445学习笔记(下)
author: 66RING
date: 2022-06-26
tags: 
- database
mathjax: true
---

# 课程笔记

## ch14 Query Planning Optimization I

> Hardest part of building a DBMS `$$$`, 人们也开始考虑使用ML做优化

基本思路: 

- 修改查询语句
	* 如规约常量, 如先选择再连接
- 与具体数据无关, 可以记录些元数据(catalog)用于优化
- 灵活cost model

执行计划的大题框架:

1. 用户输入
2. SQL重写(优化)
3. 解析
4. Binder(与元数据, catalog等), 获得逻辑计划(树)
5. 优化树
6. 逻辑计划输入优化器, 优化器根据cost model做计划

- 优化原理: 关系代数等价性: 如果量关系代数表达数的输出结果的集合相同则等价。
- **Query Rewriting**, 重写关系代数阶段的优化是与具体数据, cost model无关的

- 常用优化方法
	* Selections 选择
		+ Predicate Pushdown: 尽快做选择， 从而减少数据量。**先选择再连接**
		+ Reorder predicate: 选择性强先执行, **先过滤掉大部分数据**
		+ 分写复杂predicate: 细分选择, 方便重排
			+ $select_{A and B and C}(R) = select_A(select_B(select_C(R)))$
	* Projections 映射
		+ 除了要映射的和连接选择需要的, 其他可以去除减小元组大小
	* Joins
		+ 连接是可以交换的全部枚举找最优的方法显然行不通
		+ **维护一些内部统计数据来判断**, 可以手动更新(如SQL中UPDATE STATISTICS)也可以自动更新，可以记录如下信息:
			+ N(R): 关系R中的元组数
			+ V(A,R): 关系R中与属性A不同值的数量(方便先执行嗯区分度高的predicate)



## ch15 Query Planning Optimization II

TODO:

## ch18 Timestamp Ordering Concurrency control

> T/O concurrency control

利用时间戳来保证执行是可串行化的: 冲突就重启

定义: **如果TS(Ti) < TS(Tj)那么DBMS要保证txn执行的调度顺序与串行指定的Ti -> Tj等价**

- Basic Timestamp Ordering Protocol
- Optimisic Concurrency Control
- Partition-based Timestamp Ordering
- Isolation Levels (next lecture)

### abs

- 有冲突就重启, 反证保证执行顺序(结果)是可串行化
- local copy
	* ST单调增, 保可复读
	* "RCU"
- **Thomas Rule**
	* 另一种读写锁(共享互斥)的体现
- 瓶颈的多样的
	* OCC中存在类似RWLock中用于保护reader++的锁
- 瓶颈 -> 水平分区
- The Phantom Problem
- 优化都是出自具体分析的
	* "大数据, 小概率情景"
	* 热点与分区
	* 分区时的木桶效应


### Basic TO

> 实现无锁的事务并发控制。基本思路就是**有冲突就重启**
>
> TS(i)表示给一个transaction(txn)一个时间戳(timestamp)

- 不同的"TS"策略, 但都必须的单调递增且唯一的
	* CPU clock
	* 逻辑clock
	* 混合模式: 容易实现溢出rollback


- 每个对象X(如读取的A元组)都会打上最近一次读写的时间戳`W-TS(X), R-TS(X)`
	* `W/R-TS(x)`是针对元组对象的, `TS(Ti)`是针对事务的
	* 如每个对象都会维护一个`W/R-TS`的表
- **机制**
	* txn每个操作都会检测时间戳
	* 如果时间戳大(来自未来, 或者说被别的txn访问过), 那当前txn就终止(abort)和重启(restarting)并更新时间戳
	* 否则就能继续执行并更新该对象的时间戳为当前txn的时间戳, **然后保存对象X的一个本地拷贝**(以便防止其他读者更新read TS后无法反复读取)
		+ 有点RCU的感觉
- **Thomas rule**
	* `TS(Ti) < R-TS(X)`则终止重启
	* `TS(Ti) < W-TS(X)`则可以**忽视该write，然后继续txn**
		1. **因为有local copy**, 后续读会从local读
		2. 说明已经被人覆盖了, 而在local copy的写操作也跳过了, 所以最后不会冲突
		3. 至于WR, 因为仍然有写规则保护所以Thomas rule好得很
	* 两者结合, => "读写锁的某种变换", 某种读写锁背后的深刻哲学
- **Conflict Serializable**: 调度不使用Thomas Rule时
	* 这种策略允许存在not recoverable的事务
- **recoverable**
	* 如果一个txn依赖的txn(读了修改)都commit了, 那该txn是recoverable的
	* 不可恢复的情况:
		+ txn2的读依赖txn1写, 但是因为txn1的冲突重启了，所以txn2不可恢复
- basic T/O的问题
	* 拷贝的local的开销
	* 一个长的txn非常容易饥饿, 频繁被新的txn conflict abort

- Observation
	* **短**txn发生冲突的**概率是小**的，这种情况重试的开销是比加锁的开销画得来
- 更好的方法: 优化冲突情况


### Optimistic Concurrency Control(OCC)

- 独立workspace(local copy)
- 比对write set判断冲突
- 然后install global
- 类似**RCU**的思想

所以有如下三阶段

> 使用TS来判断RW, WW冲突

1. read phase
	1. read + write tracing read/write set of txn
	2. local copy, 以保证后续可读
2. [validation phase](#validation)
	- 在临界区内判断global view of set **还是要加锁的**
	- 将成为瓶颈
3. write phase 写回

> 有点RCU + timestamp的感觉


#### Conflict(interset)类型

- backword: 和已提交(older)的冲突
- forword: 和未提交(younger)的冲突
- 总之, 冲突判断要统一一个方向

后面章节都是forword validation分析。

**这就导致了我们无法知道对未来会不会有影响, 所以会保守abort**。e.g.

> 妙啊: 如何抽象backword和forword的底层思想?

```
T1 			T2
- READ - 

R(A)
W(A)
          - READ -
- VALI -   R(A)
          - VALI -
          - WRITE -
```

T1的validation阶段无法通过因为与T2有forword的冲突



#### validation

1. 全无交集
2. WR无交集WriteSet(Ti) and ReadSet(Tj) = nil
3. WW无交集WriteSet(Ti) and WriteSet(Tj) = nil


#### 开销分析

- observation
	* 适用于低冲突概率的场景
	* 只读

大数据 + 无偏斜 => 小概率冲突 => 但是validation的加锁也就浪费了

- 拷贝开销
- 验证和写回阶段将成为瓶颈
	* why: 因为是串行执行的 => 需要加锁
- abort的开销比加锁大, 因为txn做完后才abort
- **瓶颈**
	* 提交时要检测conflict, 需要加锁, **即使冲突率小**
	* 特别是在高并发情况下

那就引入分区！


### Partition Basic T/O

abs分区的底层思想抽象: 哈希分桶, 共享与可扩展问题

> 分布式系统的基本套路: **分区, 复制**

将数据分隔成若干无交集的子集，称为"horizontal partitions", 即shards

conflict只检测当前分区的

- 机制与条件
	* per partition queue
	* lowest timestamp优先
	* 所需partition都获得锁后txn开始
- RW无锁就abort + restart(wasteful)


#### 性能问题

- 快的条件
	* DBMS在txn执行前知道txn需要的分区
	* 大多数txn只用到一个分区

问题在于**涉及到多分区txn**

- 需要为获得所有锁而abort restart
	* **木桶效应**, 一个热点分区会拖慢整个txn


#### dynamic databases

我们目前只考虑的读和更新。删除, 插入, 更新s的情况更复杂

- **The Phantom Problem**: 你只可以在存在的tuple上加锁
	* Conflict Serializable只在对象集合大小固定的情况下保证
- 可以使用"predicate locking": 对语句加锁。但是**开销非常大**
- 可以使用"index locking", 搜索都要经过index的嘛
- 或者"repeating scan" => "反复检查"


### Isolation levels

> we may want to use a weaker level of consistancy to improve scalability

- Isolation level: txn暴露的程度
	* 将未提交的txn暴露能够获得更高并行性。不过可能代理如下隐患
	* Dirty reads(脏读)
	* Unrepeatable reads
	* Phantom reads(幻读)

|                  | Dirty Read | Unrepeatable Read | Phantom |
|------------------|------------|-------------------|---------|
| SERIALIZABLE     | No         | No                | No      |
| REPEATABLE READ  | No         | No                | Maybe   |
| READ COMMITTED   | No         | Maybe             | Maybe   |
| READ UNCOMMITTED | Maybe      | Maybe             | Maybe   |

- SERIALIZABLE, all locks -> index lock -> 严格2PL
- REPEATABLE READ, 同上, 但不需要index lock
- READ COMMITTED, 同上, 但S lock(read lock)会立即释放
- READ UNCOMMITTED, 同上, 但允许dirty read(没有S lock保护)

现代DBMS可以通过hint的方式来对具体数据做优化，如hint哪种隔离级别，hint数据库是否只读。


## ch19 Multi-Version Concurrency Control

> 并不是并发控制协议, **与并发控制协议是独立的**
>
> 并发控制"协议", 群体意识的涌现

### abs

- version chain
- version storgage
	* 插slot的, 记全表的, 记delta的
- ordering
	* o2n, 要遍历但可以gc缩短链长
	* n2o, 不用遍历但频繁修改index(导致聚簇问题)
- GC


### MVCC

- MVCC的对一个逻辑对象的多个物理版本的管理。
- 当**txn写**一个对象(如tuple), DBMS就对该对象新建一个版本, 然后end上一个版本
- 让**txn读**一个对象时, 读txn开始以来最新的一个version

**写者不阻塞读者, 读者不阻塞写者**

- 因为读者会会从某个snapshot(version)读, 不用锁, 使用TS来判断可见性
- 类似COW, 写操作更新最新的快照, 读操作读取旧版本快照
- 用于解决**脏读和不可重复读问题**: 问题的本质是读到了未提交修改
	* MVCC: 读取只能读已提交修改

- 使用timestamp维护可见性
- mvcc支持时间倒流(time-travel)

- example
	* write -> new version
	* 一个version的开始是另一个version的结束TS
	* read会从对应TS的version中读, 需要是txn所在TS时段



### MVCC Design Decisions

需要考虑的因素

- 并发控制
	* 之前的各种并发控制协议
	- 2PL
	- Timestamp Ordering
	- OCC:
- 版本管理
- 垃圾回收
- index管理

#### Version storage

- 每个tuple维护一个version链表(**version chain**), 找任意version
- Indexes总是指向"表头"
- 方法
	1. Append-Only Storage: new version插在同一表
	2. Time-Travel Storage: 旧version拷出, 新version保留
	3. Delta Storege: 只拷出所修改的, 类是log一步一步记录


##### Append only Storage

1. find slot(version是固定大小的)
2. insert
3. update pointer

- Ordering排序
	- O2N(Oldest to Newest)
		* 只需要插入表尾
		* 但是每次查询都要遍历
	- **N2O**(Newest to Oldest)
		* 差表头, 需要更新Index指向最新version
		* 查询不需要遍历(直接拿到最新)
	- "链表算法"


##### Time travel Storage

- 拷贝整个tuple
- 新version在main table
- 旧version拷贝出去, 使用指针串联所有


##### Delta Storage

> Andy: "best option"

与Time Travel Storage的区别: 只拷贝修改的属性，而不像Time travel拷贝整个tuple

类似通过日志回溯。返回指定版本就顺着指针走恢复就行了。比如先改了A1, 再改了A2。那恢复就是改回A2, 再改回A1


#### 垃圾回收

- 回收过期version: 不被任何txn可见
- 设计考虑
	* 每个对象都有version, 怎么找是个问题
	* 如何判断是过期

##### Tuple-level gc

直接扫描所有tuple, 有如下两种方法

- background vacuuming
	* 后台gc线程, 周期触发gc
	* 通过TS判断
	* 优化: 只检查dirty bitmap
- Cooperative Cleaning
	* 遍历version chain时**顺便清**
	* 需要O2N有序

##### Transaction level GC

- 每个txn维护一个read/write set, 从而能够找到tuple
- 当version不被任何txn可见时(set.find()=nil)清理


#### Index Management

> pkey = primary key

主键的Index将指向version链头

- Index修改的频率与tuple更新的频率(新version的频率)相关
- 修改主键的值 = DELETE + INSERT(重新b+树平衡)

> 多索引, 多chain如何处理

- **secondary indexes**
	* Logical Pointers
		+ 需要额外引入一层抽象(逻辑到物理)
		+ 好处是上层Index不用修改
	* Physical Poointers
		+ 直接修改物理指针
		+ e.g. 都是同一个version chain的reference


## ch20 Logging schemes

- Failure classification
- Buffer pool policies
- Shadow pages
- Write-ahead log
- Logging Schemes
- Checkpoints

- 日志全(commit) => redo
- 日志不全 => undo

- abs
	* 具体问题具体分析, 偏大概率的运行性能 vs 偏小概率的恢复性能
	* shadow page: "btrfs + 一节一page + cow策略"
	* 多个buffer轮流用, 一个进IO就用另一个
	* WAL写数据写回时机比较微妙
		+ 有日志了可以不写回, 重播就好。就是恢复时间会变长
	* steal + force抽象
		+ 真的非常抽象, 比如log也是一种单独管理数据的方式，即所谓steal
	* TODO: 两段写如何

"好像似乎都是要两段写才能保证安全, 无论分布式的还是，FS, DS的。那是否可以仅用分布式的log就行了，而不用DB本地的??distributed oranted db"


### Failure Classification

> type: 易失介质的crash, 非易失介质的crash
>
> action: crash前的措施, crash后的行动
>
> 理解crash才能保证ACID

- Txn Failures
	* Logical Errors
		+ 如除0等
	* Internal State Errors
		+ 如死锁
- System Failures
	* 软件: bug等
	* 硬件: 断电
- Storage Media Failures(无解, 只能备份)
	* checksum检测


#### Undo vs Redo

- Undo
	* 撤回一个 **未完成的txn**
- Redo
	* 重放一个**已提交的txn** (记日志但没写回完成)


### Buffer Pool Management

思考一个问题, 我们是怎么写入磁盘文件的? 至少不是Random Access地写吧? What if mmap? 但它底层也不是Random Access地写。所以, 如果全部(page)Dirty写回? 那没提交的修改呢?

- Steal Policy: 是否运行未提交的txn能"覆盖"已提交的object
	* Steal: 可以, No-Steal: 不可以
	* 一个txn能从其他txn的buffer pool里"偷"内容?
	* 不能偷就指只会写回自己的部分, copy out
- Force Policy: 是否txn的影响在提交前就会生效?
	* Force: yes, No-Force: no
	* commit前要先写回或不用
- steal|no strea + force|no force, 就有多种组合设计

思考我们平时编程是如何写回的，在看看下面的例子

```
A=3, B=5
T1: begin
T1: A=5
T2: begin
T2: B = 7
T2: commit
A=5, B=7
```

此时T2已提交, T1没提交。怎么写回呢? 平时编程的写回都是"一次一line"之类的, 很难精准跳过A。所以我们会copy out一份单独的T2的修改，然后写回。

```
T2写回
A=3, B=7
```


#### No-Steal + Force

"只写门前雪, commit前写回"

- 问题: 写回安全? 如一共写4页但只写回了2页

- 好处
	* 不需要undo, 因为txn并没有写回到磁盘
	* 不需要redo? **如写到别处**, 假设原子写回


### Shadow Paging

> 一种No Steal + Force Policy

- 维护数据库的两份拷贝: Master和Shadow
	* Master: 只保留已提交的修改
	* Shadow: 未提交的txn的修改
- 步骤
	1. 先在Shadow page中修改
	2. 提交后 **shadow master轮替**, 将shadow切换成master, 原master切换做shadow

- **"btrfs + COW的设计"**
- 一个master tree一个shadow tree
	1. shadow中修改
	2. shadow中COW(master read-only)写到别处
	3. root指针原子切换, master shadow轮替
- root切换后所有修改一定是持久化了的

- 分析
	* 回滚和恢复的方便的
		+ 不需要redo, 没有提交的是写在的别处, 要用覆盖掉就好
		+ undo: 只需要删除shadow pages重新COW即可
	* 缺点
		+ 大量拷贝开销
		+ 数据碎片话(写道别处)
		+ 需要垃圾收集(写一般没写完的)
		+ **只支持一次一write**(COW)

- Shadow Page 需要随机访问磁盘(COW), 不好利用磁盘的性质


### Write-Ahead Logging(WAL)

> Steal + No-Force

日志文件记录txn所做的修改, 从而可以读取到memory然后重播/倒放。

log的优势: 少, 因为flush往往是flush一page的, 而且steal的策略也说明会flush到其他。而WAL可以累加日志到一page后在批量写回。

1. 先内存中日志再内存中修改
2. 先写回日志再写回修改


#### WAL Protocol

- 日志将记录一些信息, 如txn id, obj id, before, after
	* e.g. `T1, A, before=3, after=4`
- 多个txn一个log buffer
	* 因为我们log buffer里记录的txn id, undo/redo时可以识别到
- **多个buffer轮流用, 一个flush中另一个就启动**
- 无论是记buffer还是写回, 都是log优先

- 日志写回时机
	* txn commit
	* txn group commit, 多利用点批处理
	* buffer满时
- 数据写回时机
	* 可以的日志写完后
	* **也可以不是**, 因为我们有日志了, 随时可以重播恢复


### Buffer Pool Policies

> 因为错误恢复是小概率事件, 几乎所以DBMS都No-Force + Steal

**tradeoff**: 重播还是搬家

- runtime performance
	* no-forece + steal: 最快
		+ 本质就是单独管理dirty page(log)
	* forece + no-steal: 最慢
		+ 如shadow page, 就需要flush所有page
- recovery performance
	* no-forece + steal: 最慢
		+ 要从单独的dirty page中一个一个恢复: 重播日志
	* forece + no-steal: 最快
		+ 直接page找回, 然后设成master
- **所以大多数DBMS关注runtime性能, 因为failure是少数情况**

TODO: **abs: 本质**


### 怎么记日志(Logging Schemes)

- Physical
	* 写回量大，但易恢复
	* object为单位
	* e.g. `diff`
- Logical
	* 写回量小, 但难恢复, 需要一步步重播。需要特别找出已经commit的修改
	* e.g. 逻辑的operation的记录
- Hybird
	* **log是page的log**
	* e.g. before: page1, after: page1'


### checkpoint

> 日志压缩
>
> next lecture

1. WAL总不能无限增长吧
2. 如果每次恢复都从"创世log"开始重播那也太久了

所以需要定期将数据写回, 然后缩短log

- WAL写入`<CHECKPOINT>`
- 写回所有已提交的数据(需要额外查找哪些数据符合要求)
- checkpoint前已提交的log就忽略
- crash发生时，根据WAL中checkpoint的记录
	* commit的redo, 没commit的undo

- 难点
	* checkpoint时会stall所有txns
	* 找未提交需要时间
	* checkpoint的频率
		+ 太短影响数据库性能
		+ 太长重播又太久

## ch21 Crash Recovery

> Actions after a failure
>
> DBMS差不多都是这种模式


- **ARIES** abs
	* WAL
		+ 先log再写回
		+ steal + no-force
		+ 重播历史: 回到crash前的一个稳定态
		+ 修改撤销: 撤销不应重播的操作
	* **Fuzzy Checkpoint**
		+ ATT, DPT, 事后再补完checkpoint
		+ 甚至checkpoint过程不主动记录ATT, DPT, 恢复阶段再扫描分析
	* **CLR**: 不单要保证正常操作的ACID, 还要保证恢复操作的ACID
	* LSN: 维护日志的"时序信息", 从而时光倒流
		+ prevLSN链表
	* 同款(raft)日志压缩技术
		+ flushed, rec, page


### Log Sequence Number(LSN)

log的metadata需要扩充, 引入LSN以**记录历史信息**。

> 上一节课中只记录的txn id, obj id, before, after等

| Name           | Where  | Definition                                    |
|----------------|--------|-----------------------------------------------|
| flushedLSN     | Mem    | Last LSN in log on disk                       |
| pageLSN        | page x | Newest update to page x                       |
| recLSN(record) | page x | Oldest update to page x, since its last flush |
| lastLSN        | txn i  | Latest record of txn T i                      |
| MasterRecord   | Disk   | LSN of latest checkpoint                      |

- 写日志
	* per page pageLSN: 记录page x修改的最新"时间"
	* flushedLSN: 已经写回的, stable的日志序号
	* 写回: page写回前需先写回日志, 即从flushedLSN一直写到pageLSNx, 使得pageLSNx <= flushedLSN

```
flushedLSN | recLSN | pageLSN |
旧         				新
从recLSN 写到 pageLSN从而实现log的增量写回
```

> 就是类似raft log中的用于trim和stable等的定位的机制

- 事务提交
	* 提交成功后记录一个"TXN-END"日志, 该日志不需要立刻写回
	* 提交后我们就可以对内存中的**日志进行压缩(trim)**


### Transaction abort

> ARIES对一个txn的**undo**, 如何undo

- 新增metadata
	* prevLSN: 事务的前一个LSN("index"链表)
	* **"链表" step by step undo**

```
012|nil T4
013|012 T4
014|nil T1
015|013 T4
```

> 显然, 新log的prevLSN就是插入前的pageLSN


#### Compensation Log Records

> CLR: 撤回上一步操作的描述。描述撤回操作本身的一种日志

**CLR也作为一种操作记录到日志中**, 保证恢复操作本身的ACID

- 新增metadata
	* undoNext: 下一步要undo的日志的LSN

"粒度细分到每个操作, 利用lazy, 一方面提升并行性, 另一方面防止crash, 需要额外metadata"

```
LSN prevLSN Tid Type  UndoNext
001 nil 	T1  Begin nil
002 001 	T1  updte nil
...
011 002 	T1  abort nil
...
026 nil 	T1  CLR-2 001 		即使在这里强制断电, 仍可以恢复, undoNext
```

"不单要保证正常操作的ACID, 还要保证恢复操作的ACID"


#### Abort Algorithm

总结以下abort的步骤

1. 写入一个Abort日志
2. 一步一步撤回
	a. 先记撤回日志: CLR
	b. 再撤回修改
3. 最后记录"TXN-END"日志

> CLR类型的日志不需要undo


### Non-Fuzzy Checkpoint

我们说checkpoint的对某一稳定状态的快照, 而为了快照操作的ACID, 我们可能会"一把大锁保平安", 而这效率的不高的, 所谓"Non-Fuzzy Checkpoint"

- Not-Fuzzy Checkpoint的影响
	* 阻塞所有新事务
	* 等待所有正在执行的事务完成
	* 写回磁盘


### Slightly better checkpoints

- Slightly better checkpoints
	* "读写锁"
		+ 阻塞会修改的事务
		+ 阻塞要读对正在修改的事务
	* 内部需要维护**ATT和DPT**

- ATT(Active Transaction Table)
	* checkpoint开始时哪些事务正在执行
	* 每个事务一个ATT项: 对事务的持续追踪
		+ txnId
		+ status: 事务的模式
			+ R for running, C for committing, U for Candidate for undo
		+ lastLSN: 链表
	* 事务提交则移除对应ATT项
- DPT(Dirty Page Table)
	* 追踪未提交事务的dirty page
	* 一个dirty page一的DPT项
		+ recLSN: 记录第一的造成page dirty的LSN


- 基本思路
	* checkpoint开始时记录ATT和DPT
	* checkpoint结束时就可以根据ATT和DPT做undo或commit

checkpoint开始时时仍然需要stall来记录ATT和DPT


### Fuzzy Checkpoint

允许active txn在写回进行时继续执行

- 新增metadata
	* CHECKPOINT-BEGIN: 标记checkpoint的开始
	* CHECKPOINT-END: 记录checkpoint过程中的ATT和DPT
	* 相当于动态记录ATT和DPT最后写回或恢复
- checkpoint完成后, 更新MasterRecord为该checkpoint-begin


### ARIES - Recovery Phase

crash后Recovery的过程

- Analysis
	* 从WAL中找到上一个checkpoint记录(通过MasterRecord)的ATT和DPT
- Redo
	* 重播所有找到的操作, 达到真正的checkpoint状态
- Undo
	* 撤回所有提交前失败的操作

- 如果DBMS在Analysis阶段又crash怎么办?
	* 什么都不做, 重试, 毕竟是收集ATT和DPT
- 如果DBMS在Redo阶段又crash怎么办?
	* 什么都不做, 重试, 毕竟redo也是某种程度的正常processing
- DBMS如何提高Redo的效率?
	* 假设不会再crash, 然后在后台异步写回
- DBMS如何提高Undo的效率?
	* 懒写回: COW, 新txn访问到时写回
	* 重写app以避免长事务


#### Analysis Phase

- 从上一个checkpoint-begin(MasterRecord)开始扫描: 找出ATT和DPT
	* 遇到TXN-END则将对应txn移除出ATT
	* 遇到commit就将txn加入ATT并标记COMMIT(补完善后)
	* 其他记录就将txn加入ATT然后标记UNDO
- 对于更新操作
	* 如果page P不在DPT则, 将P加入DPT, 然后让recLSN=LSN
- Analysis阶段结束
	* ATT就告知了哪些txn失败了
	* DPT就告知了哪些page没有写回


#### Redo Phase

- 重播历史, 包括abort和CLR
	* 当然可以做一些优化减少读写, 如写覆盖
- 从DPT中最小的recLSN开始扫描, 所有记录都redo, **除非:**
	* 影响的page不在DPT
	* 或影响的page在DPT但LSN比recLSN小(已经写回过)
- 随着redo进度更新pageLSN
- redo结束记录TXN-END日志, 将所有Commit状态的txn移除出ATT


#### Undo Phase

- 撤回未完成提交的txn
	* ATT中所有Undo状态的txn
- 使用lastLSN + UndoNext反向索引
- 同样的, 操作记录CLR日志


## ch22 Introduction to Distributed DB

- abs
	* 一致性哈希
	* "share等级"

### System Architecture

> 对外要透明, 所以节点需要一点智能, 本地不存在时自动代找

如何共享资源, 怎么收发

- Share Everything
	* 单机
- Share Memory
	* CPU通过网络共享内存
	* 每个CPU看到的Mem都相同
- Share Disk
	* "Mem as L1 cache"
	* 每个实例有自己的Mem, 但是看到的Disk是相同的
	* 因为有私有Mem, 需要 **"内存一致性协议"**
	* 对外透明, 自动代找
- Share Nothing
	* 每个实例都有自己的CPU, Mem, Disk
	* 难扩容, 难一致, 难高效


> 用户无需知道背后集群备份和分区的情况, 感觉就行操作单机节点

- 一致性与可扩展性问题
	* partition
		+ what if 多节点数据访问
		+ what if 新增节点/分区


### Design Issues

> CAP

- 需要考虑
	* 如何找数据
	* 如何在分布式的数据中执行查找
	* 如何保证正确性

- 同构节点
	* 每个集群中的节点任务都是相同的
	* 故障转移比较简单(任何都可替代)
- 异构节点
	* 节点会有自己独立的任务
	* 甚至可以引入虚拟节点充分异构


- MongoDB的例子: "数据, 逻辑分离"
	1. client向Router(mongos)发送命令(get)
	2. router向Config Server查询数据分区位置
	3. router获得到位置信息后执行, 返回


### Partition Schemes

> AKA sharding

数据可以划分到各种资源中: 磁盘, 节点, 处理器... DBMS收集各个分区的碎片, 整合成一个结果

- Native Table分区
	* 一个节点一个表
- 水平分区: 对元组s划分
	* 对rows的分区
		+ 对某个key进行hash
- 垂直分区
	* column by column
- 问题
	* 新增节点, rehash, 移动问题, 
	* **一致性哈希**

- 逻辑分区
	* 节点间可能是共享存储的, 但是是Mem私有的
	* 每个节点负责的任务不同
- 物理分区
	* 每个节点都有完整的一套Mem, Disk

- 分区, 索引, locality


### Distributed Concurrency Control

多事务多分区的并行执行。

难点在于: 备份, 通信, 故障转移, 时钟偏移

e.g. 2PL, 因为时钟的问题, 可能就会导致死锁


















