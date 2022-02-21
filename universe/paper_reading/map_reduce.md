---
title: MapReduce论文笔记
date: 2021-10-26
tags: 
- papers
- distributed
mathjax: true
---

# Abstract

分布式数据处理的设计存在很多挑战，如：大数据如何分割、如何设计并行计算、如何处理故障、负载均衡等。MapReduce是一种变成模型，使用MR来隐藏这些繁琐的细节，用户端使用MR的库，只需要考虑Map函数和Reduce函数的设计，就可以方便高效得做一些分布式的数据处理。

MapReduce的本质上也是一种分治(Divide and Conquer)思想。

# Preface

基本模型：

```
Inputfile1 -> Map -> a,1 b,1
Inputfile2 -> Map ->     b,1
Inputfile3 -> Map -> a,1     c,1
                     |   |   |
                     |   |   -> Reduce -> c,1
                     |   -----> Reduce -> b,2
                     ---------> Reduce -> a,2
```

1. Map函数对输入数据生成一些列键值对(k/v)的中间数据(intermediate)
2. MR库会将Map处理好的k/v进行分组，将key相同的归为一组`<key, itertor>`
3. Reduce对intermediate进行处理，然后整合输出(可以直接做结果，也可以再作为MR的输入)

EG: 单词统计

1. 每个输入文件用map方法解析成许多键值对: 如解析成`word: 1`
2. 用某种特定的规则将相同`Hash(键值)`的内容保存到R个中间文件，供R个reducer使用。`file-M-R`。同名单词必在一个文件中
3. 每个reducer负责`reduce`自己部分的中间文件，如reducer1负责收集`file-1-1, file-2-1, ...`
4. reduce阶段结束参数R个中间文件，最后再合并


- 可以抽象成一下六个过程，这里进行一个类比
	1. input
		- mass data
	2. split
		- 分任务规模 
	3. map
		- 执行任务的具体处理
	4. shuffle
		- TODO 
	5. reduce
		- TODO
	6. finalize
		- 仍然可以给到MR

# 正文内容

## goolge的MR结构

- Map: map to key values for each input file. Having nMap mapper
- mapper结束向master发送信息，master要记录位置和大小，任何转发给reducer
- Reduce: collect all the same key and some other work (like collect and count word)

```
Input1 -> Map -> a,1 b,1
Input2 -> Map ->     b,1
Input3 -> Map -> a,1     c,1
                  |   |   |
                  |   |   -> Reduce -> c,1
                  |   -----> Reduce -> b,2
                  ---------> Reduce -> a,2
```

大体流程如下：

1. 首先将输入文件切分成多份(split)，然后启动用户程序，并拷贝到多台机器上运行
	- 方便Map worker的分布式并行处理 
2. 选取一个程序作为master，多个worker，这些worker将会被master调度执行map和reduce任务
3. map worker: 
	a. 读取指定的一份split
	b. 解析k/v，传递到用户定义的Map函数进行处理
	c. Map函数生成intermediate在buffer中
4. buffer中的数据会定期地写入本地磁盘
	a. 由分区系统确定如何分区成R regions
	b. 分区完成后告知master分区信息，再有master通知reduce worker
5. reduce worker:
	a. 根据master给出的位置信息读取(通常master会调度数据在本地或就近的机器做reduce)
	b. 排序，将相同k的数据归到一起
6. reduce worker遍历排好序的intermediate，传递给Reduce函数
7. 所有map reduce执行完成后master返回用户程序


## 设计细节

### master data structure

- Worker informations
- Regions informations

master需要调度work，分发工作范围，因此需要记录分布在其他机器上的worker的信息，同时也需要记录intermediate的位置信息。

### Fault Tolerance

- worker failure
	1. master定期检查
	2. 以完成的worker和failure的worker都变为idle待命
	3. master知道该重新执行什么内容，调度idle worker处理
- master failure
	* 定期check point，可以从check point恢复
	* google具体问题具体分析，master不太可能failure，所以直接简单的客户端重新MR
	

### Locality

网络带宽是相对稀缺的资源，因此reduce读取"远程的local disk"的过程尽量发生在本地，或者尽可能的就近。

得利于底层的GFS，master可以找到含有"local disk"的replica，来做reduce

1. 一个数据会有多个replica(map后的local disk)
2. map处理后每份(Region)在16MB～64MB，GFS的一个chuck(保持locality)
3. 这样一来，利用GFS的replica，可以找到含有replica的work执行reduce


### 任务粒度

map会生成M个Region，reduce生成R份output，master需要对这些内容和各个work进行组织。所以什么设计worker数量，切分的粒度就需要讲究。

1. O(M + R)个worker可以调度，需要在内存中记录
2. O(M x R)个状态需要记录(中间结果集)

google具体问题具体分析，O(M x R)实际占用内存是可以接受的，所以看看其他方面可以如何优化：

1. M个Region，每个应在16MB～64MB
2. R个，几倍于worker


### ⭐Backup Tasks⭐

受到木桶效应的影响，最后完成的几个worker("straggler")往往是整体处理时间长的原因。出于各种原因：磁盘、内存、CPU等损坏等，这些worker大概率会远远慢于其他已经完成的worker。

因此在MR操作快完成时(设置一个阀值)，master就会调用backup task执行，执行的内容就是那些正在执行的straggler执行的内容。任何一队完成则MR操作结束。

实验证明，这个操作使得完成时间大幅提高：不做backup tasks平均耗费44%更长的时间。


## 细节处理

设计好一些基础设施，如：

- 预留partion system等，提高灵活性
- 设计Combiner，因为要从分布式的系统中将数据发给reduce，适当将数据整合再发提高网络利用
- 支持多种输入格式(输入文件的切分方式)
- 设计debug设施：允许单机顺序执行模拟分布式并发
- 用户友好的状态信息显示等

- 等待阶段，任务发完了还有worker
- crash判断
	* crash后发来两个complete
	* crash后发来两个complete, 进入下一个阶段后第二个complete才来
- 使用(buffer/unbuffer)channel做同步真的很容易阻塞

# 实验设计

# 亮点与不足

## 亮点

- [Backup Tasks](#⭐backup-tasks⭐)
	* 秒秒秒，体会
	* TODO：抽象一下背后的道理?
- 基于GFS这种global file system
	* 屏蔽了数据文件"同步"的细节，多机用着像单机
	* 充分利用底层服务


# What's more

# 思考

- 论文组织
	1. abs: 不拖泥带水地介绍的核心内容(MR)
	2. introduction
		- 问题驱动(提出问题)
		- 基本设计思路
		- 引出下文(各个章节的summary)
	3. 正文组织
		- 基本模型
		- 设计细节
		- 细节精炼
	4. 实验设计：TODO
		- 实验设计
		- 成果展示
	5. related work
	6. conclusions
	7. ref/acknowledge
