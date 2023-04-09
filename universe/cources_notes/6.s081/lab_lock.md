---
title: MIT6.S081 lock lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# Locks

重新设计代码，提高并行度，降低锁竞争。降低锁的竞争会涉及数据结构和锁策略的改变。

提升xv6内存分配和块缓存的并行度。

> Section 3.5: "Code: Physical memory allocator"
> Section 8.1 through 8.3: "Overview", "Buffer cache layer", and "Code: Buffer cache"


## Memory allocator

`user/kalloctest`压力测试内存分配器: 三个线程同时申请释放大量内存, 大量调用`kalloc`和`kfree`。`kalloc`和`kfree`都会竞争`kmem.lock`锁，`kalloctest`会打印申请不到锁(因为被别人占有)的次数，以此粗略估计竞争情况。

```
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: bcache: #fetch-and-add 0 #acquire() 1260
```

`#fetch-and-add`表示acquire时锁被别人持有的数量，`#acquire()`表示锁竞争的粗略值。

导致锁竞争的根源是因为`kalloc`使用锁保护单一的free list。你需要重新设计内存分配器以避免单一的锁和空闲队列。

一个基本的想法是为**每个CPU维护各自free list**，每个list用自己的锁。这样申请和释放就能并行地处理。主要的困难是如何处理一些CPU内存空了，但是其他没空的情况。这时需要从其他cpu获取内存，这就涉及到互斥，但是这种情况不那么频繁。

这个lab要实现per-CPU freelist和内存不足时在CPU间迁移的情况。你的所有锁都要以"kmem"为前缀，需要在`initlock`中初始化锁。然后运行`kalloctest`, `usertests`和`sbrkmuch`测试。


### hints

- 可以参考`kernel/param.h`中的常量`NCPU`
- 注意为调用`freerange`的CPU分配所有空闲内存，只需要在一个hart中初始化分配器即可
- `cpuid()`可以获取当前cpu号，但是需要**注意关中断**才能安全使用
	* 可以使用`push_off()`和`pop_off()`关开中断
- 可以参考`kernel/sprintf.c:snprintf()`来学习字符串格式化方法。虽然所有锁都叫"kmem"也行


### result

- `push_off()`, `pop_off()`关开中断
- 多核cpu下的初始化情况(main)，每个cpu都要设置内核页表，设置中断向量。其余可以只在一个cpu中初始化。详见`main.c`
- 可以一次"偷多页"进一步减少锁竞争


## Buffer cache

这个lab将独立于上一个实验。

多处理器对文件系统的使用是十分频繁的，就会导致`bcache.lock`的大量竞争。这个锁用于保护`kernel/bio.c`的disk block cache。

`bcachetest`会创建几个进程反复地读取不同文件来引发`bcache.lock`锁竞争。`bcache.loc`用于保护"list of cached block buffer", `b->refcnt`和cached blocks的id(`b->dev`和`b->blockno`)

修改block cache，使`#acquire`的值接近0。理想情况下所有block cache相关的锁都会是0, 最坏情况不应该大于500。**修改`bget`和`brelse`**使得对不同块的并发的查找和释放不会冲突。你需要保证每个block最多一个cached。

你的所有锁应以"bcache"为前缀。降低block cache的竞争要比kalloc难，因为bcache是会在多个进程间共享的。对于`kalloc`只需要给每个cpu一个内存分配器就可以解决大多数竞争，而在bcahe中这是不够的。

推荐使用哈希表来查找cache中的block号，然后单独为每个hash bucket加锁。

如果在下列情况中，你的实现出现了锁竞争，那也是没问题的。

- 当两个进程并发使用同一个block number
- 当两个进程并发cache miss并需要找一个未使用的块
- 当两个进程并发使用块导致冲突，而冲突是你划分block和lock的策略导致的
	* 如两个进程使用的块有相同的哈希值，但你应该调整并尽量减少(如增加哈希表大小)

`bcachetest`的`test1`会频繁使用不同的块，而少使用缓存的块。


### hints

- 阅读参考书block cache部分内容(8.1-8.3)
- 可以使用固定大小的哈希表桶。可以设置一个**质数大小**(e.g. 13)以减少冲突
- buffer不存在时的(查表和分配)应该是不可分的操作
- Remove the list of all buffers (bcache.head etc.) and instead time-stamp buffers using the time of their last use (i.e., using ticks in kernel/trap.c). With this change brelse doesn't need to acquire the bcache lock, and bget can select the least-recently used block based on the time-stamps.
	* 我并不理解它所谓基于时间戳做LRU是什么意思，我觉得在release时插入队首，bget时移到队首，那么就会队首新队尾旧。recycle时直接从队尾找就行(这也是原LRU策略，只不过它更抽象，它是relse时才作为LRU插入队首)
- cache miss时`bget`会挑选一个没人用的buffer recycle。可以尝试序列化的查找
	* **之所以叫recycle**是因为block释放`bufrn`为0时不会清空buf内容，而是LRU保存缓存，后面再次用到就不用重新传输数据。当cache miss时才将buffer分配给其他block这时内容才会彻底更改
	* It is OK to serialize eviction in bget (i.e., the part of bget that selects a buffer to re-use when a lookup misses in the cache).
- 你的方案有时可能会同时持有两个锁，如在挑选过程中你可能会同时持有bcache lock和bucket lock。所以要小心死锁
- 当新建缓存时，你可能需要重其他bucket移入才能分配(recycle)。这时如果源bucket和目标bucket相同，则可能后导致死锁(多重加锁)
- 调试技巧：实现bucket lock但保留`bget`开始和结尾的用于序列化的全局`bcache.lock`。当正确且没有数据竞争时，就可以删除全局`bcache.lock`然后进一步处理并发问题了
- 可以使用`make CPUS=1 qemu`来单核调试


### result

- hash table bucket
	* use a prime number of bucket to reduce the likelihood conflict
- **经历了指针提前释放bug**，以下是我修改这个bug的心路历程
	1. 因为`initlock`的命名仅仅是改变指针指向，而如果函数中的使用的是`buf[]`保证字符串，则在函数返回时会被释放了。
	2. 需要使用`char *`，而这就涉及到内核空间的内存分配，懒，所以索性用全局变量了。但是这样就又出问题，还是指针赋值问题，`lock->name`都指向同一个buf，而这个buf只有它最后应用的内容
	3. 最后索性全都只叫`kmem`, `bcache`, `buffer`了
- **之所以叫recycle**是因为block释放, `bufref`为0时不会清空buf内容，而是LRU策略保存缓存内容，后面如果再次用到就不用重新传输数据。当cache miss时才将buffer分配给其他block这时内容才会彻底更改
- 需要注意的是从别的bucket偷buffer时需要立刻移动到当前bucket，因为后续可能会复用(即`refcnt++`的情况)
- 本质上还是**"使独立"**的思想，根据不同的哈希值哈希到不同的相互独立的桶从而减少竞争

- block cache感悟
	1. LRU cache 链表结构体现了局部性原理
	2. 块设备先缓存到内存再从内存中使用，体现"速度差"原理
	3. 除了第二点外，block buffer还使**高效复用block**成可能。(即不同进程对同一个block的访问可以使用同一个buffer)
- 分桶
	1. 外设块设备相当于共享资源，分桶锁可以有限减少锁竞争


## xv6 fs

```
| boot   | superblock | log       | inode | bitmap | data |
| block0 | block1     | block2... | ...   | ...    | ...  |
```

- superblock是由`mkfs`构建的

### buffer cache layer

job of buffer cache layer

1. 同步磁盘访问以确保一个block在内存中只有一份拷贝
2. 缓存常用block，减少慢速的磁盘读写(`bio.c`)

主要接口

- `bread`获取内存中的block
- `bwrite`将内存中的block写回磁盘
- `bread`返回一个上锁的buffer，然后`brelse`释放per-buffer锁

buffer cache是固定大小的，让cache miss时需要LRU换入换出。










