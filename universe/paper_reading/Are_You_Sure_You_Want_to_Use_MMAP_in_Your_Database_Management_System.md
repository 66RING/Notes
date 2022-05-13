---
title: Are You Sure You Want to Use MMAP in Your Database Management System?论文笔记
author: 66RING
date: 2022-05-10
tags: 
- papers
- database
- mmap
mathjax: true
---

TODO review

# Are You Sure You Want to Use MMAP in Your Database Management System?论文笔记

# Abstract

> 我要重启window时它居然自动更新了

mmap看起来好用，是因为其让一些操作"透明"，而这个"透明"将导致很多严重的问题。就是说OS偷偷执行的, 通用的服务带来不可控性。即使熟悉OS的各种操作仍然可以把mmap把持住，但是这无疑又引入了额外的复杂度，就忘了使用mmap的初衷。

- keyword
	* 不可扩展性问题
	* 只有应用程序自己自动自己需要什么，OS为了"通用"那就会失去一些关照
	* 缓存污染
	* 自动写回太拉了：重启window时系统自动更新了


## 小技巧get

"左右手轮流，减少了拷贝开销"


# Preface

很多人会喜欢使用`mmap`来方便的实现buffer管理，主要的优势有两点：

1. mmap避免了很多显式的系统调用(read/wirte)，操作透明，好用
2. **用户直接访问page cache**，而不用拷贝到用户空间后再使用

这个透明和直接访问page cache将是整个mmap的万恶之源。


# 正文

## mmap和一些概念的介绍

> 有备而来，无懈可击。你想到的我也都想到了

### TLB shootdown

OS在做页面的淘汰时还需要让TLB无效化。但是 **为了保证缓存(TLB)的一致性，OS需要进行很多的CPU间通信** ，因为TLB是每个CPU独立的，CPU间需要非常大的开销达成缓存一致，这个使用开销巨大的处理器间通信(inter-processor interrupt)来更新TLB就叫作 **TLB shootdown**

缓存一致性的问题的原理，可以看我这篇管理[缓存一致性协议的文章](https://github.com/66RING/Notes/tree/master/universe/os/MSI_coherence_protocol.md)


### mmap

`mmap`是一个能将可以将外存文件映射到内存，然后用户可以使用内存访问的方式来操作文件的内容，文件的内容的映射，写回由操作系统完成。

其中两个映射方式：

1. `MAP_SHARED`: 任意写都会触发对底层文件的写回
	- 无疑会导致不想IO时自动做了IO
2. `MAP_PRIVATE`: 创建一个COW映射给调用者


### madvise

显式告知(hint，就是使用不同的参数啦)操作系统访问模式，这样OS就可以采用不同的预取策略等。"OS 提供很多很多feature"。

- `MADV_NORMAL`: 每次从外存预取128KB
- `MADV_RANDOM`: 每次只取需要的page
	* larger-than-memory的OLTP情况
- `MADV_SEQUENTIAL`: 适用OLAP


### mlock

将一个page固定(pin)在内存，即不会收到淘汰机制的影响。但是操作系统仍然会对pinned的page做**自动写回**。


### msync

显式的让操作系统将一段内存中的数据写回到外存中。保证持久话。


## mmap的问题

> 这些问题如果给到操作系统开发者他们会说：ok，我们可以该，加点支持

1. 事务安全性
2. IO stalls
3. 错误处理
4. 性能问题

其中前3点一些精英可能会找到OS提供的各种方法来修复，但这无疑引入了额外复杂度，而忘记当初使用mmap图简单的初衷。

ok行，复杂就复杂一点呗。但是第4点是无解的，除非对操作系统做非常多的修改。


### 事务安全

> 不想IO的时候IO了，window重启的时候自动更新了

因为mmap依靠OS做内存管理，OS就可能在任何时候将页面写回。一方面，试想一下，自动写写到一半断电了。另一方面，会引入白费的IO操作，即IO后rollback，那开销就浪费了。

导致事务不安全的本质原因是"原地修改"，文章提供了三种在"副本修改再写回"的解决方案：


#### OS Copy-On-Write

利用`mmap`的`MAP_PRIVATE`参数，修改(update)会发生在副本，当在副本完成事务，WAL(日志)记录完全了就可以拷贝会primary了。

不过这样有两个主要问题：

1. 需要额外维护副本中的哪些修改需要写回，并保证所以修改都完全写回了
2. 保留完整的拷贝很浪费空间
	a. 我们可以使用`remmap`缩小副本占用的空间，但是这就是又增加了问题1的开销


#### User Space Copy-On-Write

手动将修改涉及到的page拷贝到用户空间，然后副本修改，记日志，然后写回。占用小了，但引入了拷贝开销。


#### Shadow Paging

> 主从轮流当，减少了拷贝回主的开销

同样的，维护一个副本(可以是backed by mmap)。修改发生时，复制涉及到的page到shadow page。

提交时将shadow pages的内容写回外存，然后**让原本的shadow变成主，原本的主变成新的shadow**

**妙啊**


### IO Stalls

**mmap不支持异步！** ，如果碰到了不连续的访问，那么中途对外存的访问将拖慢整个系统的执行。传统的read/write管理buffer pool就支持异步IO。

另外，考虑 **缓存污染/"不合理淘汰"** 问题，即OS自动淘汰page，非常不可控。

总结一下上面问题的本质：访问任意page都可能导致不不想要的IO stall

解决方法有几个：

1. 对于缓存污染或者说"不合理淘汰"问题，可以使用`mlock`固定(pin)page
	- 问题在于OS通常会显示pin的page的数目
2. 使用`madvise` hint 操作系统，"启动不同的模式"
	- 如OLTP数据库可以开启`MADV_SEQUENTIAL`
	- 不过能控制的还是太少
3. 不予无异步方面，可以在子进程/线程中做预取或read/wirte，这样就不会阻塞主线程


解法很多，但是复杂度上去了哦


### Error Handling

> IO时间的不确定，保守起见导致频繁checksum检查

DBMS中会为每个page做一些checksum的检测以防止IO时的数据错误。但是对于mmap，因为它导致了频繁的IO，而且 **我们明不知道它什么时候做了IO**，这就导致了我们每次都要对page做checksum检测。

还有就是我们一般会 **先检测checksum再写回**，但是如果是mmap，因为我们不知道它什么时候写回，所以就会导致checksum检查前就被写回了。


### 性能问题

> [缓存一致性问题](https://github.com/66RING/Notes/tree/master/universe/os/MSI_coherence_protocol.md)
>
> [可扩展性问题](https://github.com/66RING/Notes/tree/master/universe/os/noscalable_lock_and_solution.md)

最后一个，也是最无解的问题就是使用mmap的性能问题: [不可扩展问题](https://github.com/66RING/Notes/tree/master/universe/os/noscalable_lock_and_solution.md)

不可扩展问题

三大瓶颈：

1. 页表竞争
2. 单线程做的page淘汰
3. [tlb shootdown](#tlb-shootdown)

操作系统可以解决前两个，但是TLB shootdown就难了。tlb shootdown：缓存为了达成一致，需要进行额外的通信开销，CPU越多这个开销越大，因为CPU间需要更多的通信，这样一来 **原本一个cycle的指令就翻了10倍甚至1000倍**


## 亮点

- 做好铺垫和介绍，让文章更完成
- 提出问题，再给出解决方案，最后引入无解的问题，增加文章的可信性






