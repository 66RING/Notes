---
title: CMU15-445学习笔记
author: 66RING
date: 2022-06-06
tags: 
- database
mathjax: true
---

# Coding

Google c++ style guide

检查丑代码:

```
make format
make check-lint
make check-censored
make check-clang-tidy
```

# Hybrid OS

- 操作系统中的启发(可能有用)
	* fs: 分段多级索引

# Mind Map

- Storage
	* 缓存管理策略
		+ lru/mru/clock
	* 磁盘文件优化
		+ heaps
		+ linked list
	* 内存布局
		+ slotted page
		+ log-structured
- Hashing
	* 静态哈希
		+ 线性探测，罗宾汉，布谷鸟哈希
	* 动态哈希
		+ Extendible Hashing
		+ Linear Hashing
- Tree Index
	* B+树
		+ 插入，删除，合并，分裂
		+ 与B树的区别
		+ 锁方案
			+ Crabbing/Coupling
	* 基数树(Radix Tree)
- Sorting
	* 外排
		+ 两路，多路(general)
	* 开销分析，计算
	* buffer数的影响
- Joins
	* 多重循环(Nested Loop)
	* 基于排序聚合(Sort-Merge)
	* 基于哈希
	* 不同开销分析，各自适用情况
- Query Processing
	* Processing Model
		+ process, thread per work, pool. 优缺点
	* Parallel Execution
		+ Intra vs. Interwork


# 课程笔记

## ch1 history

## ch2 SQL用法

- 表的增删改查创等
- having, group by, 子查询，集函数等

## ch3 数据存储1: 数据结构

- page为单位操作，page除了4KB还可以其他多种方式
- mmap为什么不好，不可控
- 如何组织数据存储的方式
	* 简单的："数组"，但是在内部删除增加时存在问题
	```
	| cnt: 3 |
	|--------|
	| id1    |
	| id2    |
	| id3    |
	
	delete id2
	| cnt: 2 |
	|--------|
	| id1    |
	|        |
	| id3    |
	```
	* slot，标记的方式
	```
	page 
	|-----------------------|
	| slot1 | slot2 | ...-> |
	|                       |
	|                       |
	| <- ...| data2 | data1 |
	```

## ch4 数据存储2: 存储模型

- 关系型，非关系型...
- **OLAP**
	* case: 指令复杂，很多集函数，子查询，LIKE等运算
	* OLAP场景多是读多复杂性高
	* 列存储结构
- **OLTP**
	* case: 指令简单，一条执行就简单的增删改查等
	* OLTP场景多是写多复杂性低
	* 行存储结构

e.g. 下面是一个OLAP的例子，只涉及到`lastLogin`这个属性，但是如果数据库只是简单的笛卡尔积，将带来大量的空间浪费。

```sql
SELECT COUNT(U.lastLogin)
	EXTRACT(month FROM U.lastLogin) AS month
	FROM useracct AS U
	WHERE U.hostname LIKE '%.gov'
	GROUPY BY EXTRACT(month FROM U.lastLogin)
```

所以对于OLAP场景，多使用列存储的结构，即每列都附带上key，列直接通过相同的key联系在一起：

```
列存储
id1 |col1|  id1 |col2|  id1 |col3| 
id2 |col1|  id2 |col2|  id2 |col3| 
id3 |col1|  id3 |col2|  id3 |col3| 

行存储
id1 |col1|col2|col3| 
id2 |col1|col2|col3| 
id3 |col1|col2|col3| 
```

OLTP，OLAP，HTAP：

```
complexity  ^
            |         OLAP
            |    HTAP 
            | OLTP
easy        |------------>
             read       write
```

> 那有没有基于矩阵的存储呢?


## ch5 Buffer pool

缓冲池的优化方案

1. Multiple Buffer Pools
	- 各用各的buffer pool，不相互污染
2. Pre-Fetching
3. Scan sharing
4. Buffer Pool Bypass

- Conclusion
	* DBMS内存管理要做得比OS好，毕竟定制
	* "通用的不要，我们按需定制"
		+ e.g. os page cache, buffer pool bypass
	* 各种方向的策略都是学问
		+ 淘汰策略
		+ 预取测略
		+ 分配测略


### Multiple Buffer Pools

per-database buffer pool和per-page type buffer pool等，想方设法 **让共享资源独立** ，这样可以 **减少锁竞争和增加局部性**


### Pre-Fetching

根据查询策略可以由不用预取方案。如顺序读写，就可以预取顺序后续的页。再如树形结构的扫描，可能需要一些方式来判断哪些页不是叶子节点，从而只预取保存了数据的叶子节点。


### Scan Sharing

假设有Q1, Q2两个扫描。Q1执行到一半然后Q2开始执行，都是对同一个表的扫描。这时 **Q2可以attach到Q1** ，即从Q1的位置开始扫描，最后Q2再自己扫描把前半段补上。这样很明显就容易污染缓冲区。

不过需要注意`LIMIT`等的情况，需要额外的处理。


### Buffer Pool Bypass

不使用"自动淘汰测略的共用Buffer Pool"， **自己独立申请内存来做buffer** 。如在扫描大型文件时，buffer pool是不够大的，那么一趟下来虽然自动buffer了，但是又被后续扫描覆盖了，从而一次都没hit到。

又比如我们不希望临时表等文件污染buffer pool。


### 越过操作系统的Page Cache

我们知道操作系统会做Page Cache，但是os做的是受到通用性等约束的。我们不希望其他程序污染，或者我们要根据需求实现自己的缓存方案，我们可以使用`O_DIRET`标记来跳过os的page cache。


### LRU

- page用上次访问的timestamp标记，淘汰时取timestamp最老的淘汰
	* 所以pages要根据timestamp有序存放(最小堆)
	* "为何不用链表?" 出于局部性的考虑?
- **Clock** 策略：二次机会
	* 一个page的循环数组，每个page由ref标记最近是否有使用，一个指针单向遍历
	* 如果指向的page，ref=1说明最近有使用，ref置0
	* 如果指向的page，ref=0说明最近没使用或至上次使用已经不用一段时间了，淘汰之

不过这些"传统的LRU"方案都存在"sequential flooding"问题。如在扫描大型文件的时候，你的buffer有限，e.g.:扫到page3时buffer1要淘汰，到page4时之前page1和page2的cache都不复存在。另外，这些cache **往往只读一次就不用了** ，非常污染我们的缓存策略。

```
buffer  page
1        1
2       2
        3
        4
        5
```

**改进**: 

1. LRU-K
	- 不单记录timestamp，还记录前K个操作对该page的使用频率
2. Localization
	- 每个query到设置缓冲区，最小化缓冲区的污染
		* e.g. small ring buffer for private query
3. Priority hint
	- DBMS应该知道页面的内容，所以知道哪些页面的内容比较重要


- 其他优化: Background writing
	* 周期性写回脏页，而不是cache满时要换出时再写回
		+ 需要注意先记录log再写回


## ch6 Hash Table

- 静态
	* Linear probe hashing, 线性探测哈希
	* Robin hood hashing, 罗宾汉哈希
	* Cuckoo hashing, 布谷鸟哈希
- 动态: 扩容的影响尽可能小：resize on demand
	* Chained hashing
		+ 平均距离比线性探测好，但是大量指针就不能利用好局部性了
	* Extendible hashing
		+ 本质上是数组的扩张，局部性不错，只是每次翻倍可能会浪费空间
	* Linear hashing
- 溢出
	* 溢出策略自定，具体问题具体分析，可以根据avg也可以根据max...

- Conclusion
	* 想办法减少线性探测的平均长度(e.g.罗宾汉, 链式)
	* 哈希表是存储引擎从page中获取data的一种方式，另一种是树
	* **动态哈希思想**: 
		+ 分裂的影响尽可能小：**resize on demand**
			+ 如Extendible hashing和Linear hashing


### Linear probe hashing

当哈希同一个slot时，如果该slot空则使用，否则查找线性下一个，直到找到空slot。

**需要注意删除的情况** ，因为的线性查找下一个，如果直接删除会错误以为key不存在，所以删除元素的方法是逻辑删除：标记删除。

> 相同key不同value的处理方法
> 
> - key-values
> - (key-value)s


### Robin hood hashing

罗宾汉哈希的思想是尽可能降低平均线性探测的时间。

- rick key: (相对poor key)insert的位置距离最优的位置(即哈希立即得到的位置)的距离 近
- poor key: 同理rick key，距离较 远

所以方法就是：另外维护一个"距离"的项，插入是比对"待插入元素"当前距离和"已插入元素"当前距离，poor key会得到该位置。罗宾汉劫富济贫


### Cuckoo hashing

> 鸠占鹊巢

使用多个哈希表，每个哈希表使用不同的哈希函数。每次插入同时尝试所有哈希函数，如果直接是空位就插入，否则使用另一个哈希函数。如果所有表都已被占用，则淘汰当前占用的元素。最后该被淘汰的元素再用上述方法找到它的位置。

需要注意的是 **可能会出现死循环** ，所以对哈希表大小的设计有要求，不过查表是O(1)的


### Chained hashing

上面提到的属于静态哈希表，哈希表大小固定，每次哈希表的resize需要所有元素重新哈希。

动态哈希表一定程度上解决了这个问题，因为是"resize on demand"的

Chained hashing就是相同的`hash(key)`放到同一个桶(bucket)中，每个桶是一个page，溢出则分配新桶链到后面即可


### Extendible hashing

"数学原理"：大range分小range

1. 初始两个大range: `0xxx`和`1xxx`
2. 溢出后将一个range细分：如分为了`0xxx`，`10xx`和`11xx`，那么对于原来的大range`1xxx`就多了一个page来存储

Extendible hashing比较复杂。具体如下：

- 一个global count表示哈希时考虑几bit(其实也就是所有local count的最大值)
- local count表示哈希到该页用了几bit(其实也就是分裂的次数，1开始表示0次)
- 如果一个表溢出，则分裂成两个表，这这两个表的local count都是原local count+1
- 如果分裂后local count大于global count，则global count++，索引数组 **翻倍**
- **每次分裂需要重新映射** ，其实也就分裂掉的page要重新映射

思想的这样子的：一个二进制数翻倍后会得到两个不相关的数，如原来只考虑一位时，`1`能够索引一个page，翻倍后得到`11`和`10`用于索引两个表。分裂对于其它表没有影响，比如`0`，因为它管理的范围仍是`0xxx`，那让所有`0xxx`都索引到该page即可，溢出后再分裂。


```
global:1
   ┌─┐        ┌────┐
0xx│ ├───────►│0xxx│local:1
   ├─┤        │0xxx│
1xx│ ├─────┐  └────┘
   └─┘     │
           │  ┌────┐
           └─►│1xxx│local:1
              │1xxx│
              └────┘


global:2
   ┌─┐        ┌────┐
00x│ ├─┬─────►│0xxx│local:1
   ├─┤ │      │0xxx│
10x│ ├─┼───┐  └────┘
   ├─┤ │   │
01x│ ├─┘   │  ┌────┐
   ├─┤     └─►│10xx│local:2
11x│ ├───┐    │10xx│
   └─┘   │    └────┘
         │
         │    ┌────┐
         └───►│11xx│local:2
              │11xx│
              └────┘

```


### Linear hashing

**数学原理**

```
hash1: key % n = M
hash2: key % 2n = M 或 M + n
```

所以可以将 **桶M分裂成桶M和桶M+n** 。我们有办法知道哪些桶分裂，哪些桶没分裂，对于没分裂的桶我们使用hash1, 分裂的桶使用hash2。 **这样的resize on demand就解决了直接扩容导致的瞬时大量迁移**

哈希表维护一个指针，用于标识下一个要分裂的桶。 **任意** 桶溢出将导致 **split pointer指向的桶** 分裂，然后指针下移。

**注意** 分裂的是指针指向的桶。

会需要使用多个哈希函数来寻找key对应的桶。详见后文。

溢出判断方法可以是：超过容量溢出或超过某AVG溢出。下例假设桶容量5，超过则溢出

| split ptr | 桶id | 已存的key         | 溢出key |
|-----------|------|-------------------|---------|
| =>        | 0    | 4, 8, 12          |         |
|           | 1    | 5, 9              |         |
|           | 2    | 6                 |         |
|           | 3    | 7, 11, 15, 19, 23 |         |

- 第一个分裂的桶从0开始
- hash1(key) = key % n
- hash2(key) = key % 2n

现在 **插入**key=27: 

1. 桶3发生溢出
2. 指针指向的桶：桶0分裂
3. 桶0的内容用hash2重新映射
4. 指针指向下一个桶

| split ptr | 桶id | 已存的key         | 溢出key |
|-----------|------|-------------------|---------|
|           | 0    | 8                 |         |
| =>        | 1    | 5, 9              |         |
|           | 2    | 6                 |         |
|           | 3    | 7, 11, 15, 19, 23 | 27      |
|           | 4    | 4, 12             |         |

查找：

1. 如果hash1得到的值大于等于split ptr指向的桶id，则所得即为目标桶
2. 如果hash1得到的值小于split ptr指向的桶id，则使用hash2找到真正的桶
	- 因为split ptr之前的桶已经分裂了，而这个分裂是使用hash2重新映射的
3. 当原始桶都分裂过了(即0~3)，则根据那时的大小开始新的hash1，hash2循环

删除：只有最后的桶为空是才会缩小


## ch7 Table Indexs(B+tree)

> - table index: 维护一段有序或以某种方式组织的table的子集以提高访问效率
> - DBMS要保证table和index是同步的
> - B+树可以有大于两个的孩子
> - B+树提升读写效率

[B+树可视化](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

- B+树特性
	* 完全平衡，深度相同
	* 每个叶子节点是 **half-full**的
		+ 但是merge的成本很高，有些DBMS会不做merge
	* 如果中间节点有k个key，那它有k+1个孩子
	* 叶子节点之间使用姐妹指针链接
		+ 因为DB中常会有顺序select等操作，姐妹指针链接后就 **不用像B树那样频繁回溯的遍历了**
- 与B树区别
	* B树 **非叶子节点会保存value** 。B+树 **只在叶子节点保存value** ，中间节点只保存key
	* 所以 **B树遍历需要大量回溯**
	* B+树矮胖，不过好像访存复杂度和B树一样，都是二分查找。只不过是B+数局部性更优: 包括节点内二分，不用回溯

- conclusion
	* 妙不可言，妙不可言
	* 时间复杂度，空间复杂度， **局部性复杂度**
	* 尽可能利用好一次磁盘连续访问
		+ sibling pointer
		+ node size
		+ why not B tree(回溯, 连续)
	* 不要忘了局部性
		+ why not key pointer
		+ key value分开存(索引值保存key)
		+ 内部偏移: key map/indirection
	* 有序的特性
		+ 二分查找
		+ Interpolation
		+ 前缀压缩
		+ 后缀截断(inner node, 够区分就可以了)
	* trade off
		+ 合并
		+ 重建
		+ 热点节点

```
       ┌─┬──┬─┬──┬─┬─────┐
       │ │5 │ │9 │ │     │ Inner Node
       └┬┴──┴┬┴──┴┬┴─────┘
        │    │    │
  ┌─────┘    └──┐ └───────┐
  │<5        <9 │         │>=9
  │             │         │
  │             │         │
  ▼             ▼         ▼
 ┌─┌─┬─┬──────►┌─┬───┬───►┌─┬───┐
 │1│3│ │       │6│7  │    │9│13 │ Leaf Node
 └─┴─┴─┘◄──────┴─┴───┘◄───┴─┴───┘
         Sibling
         Pointer
```


### Nodes

B+树每个节点都是`key-value`数组( **一个page** )，只是叶子节点和中间节点的value不同。 **不过实际中不会这样简单的一个数组** 。

常用的是key，value分开存储。key一个数组，value一个数组。**原因如下：**

- key/value/key-value的大小是可变的
- 扫描/搜索时用不到value，只要key不想value污染cache(我们对cache真的非常小气)


### B+树插入

1. 找到叶子节点L(key在某个区间)
2. 插入数据，保证依旧有序
	a. 如果L空间够大，结束
	b. 否则L分裂成两个节点
		i. 将(有序)中位数key拷贝到上层去做所以
		ii. 重新分配被分裂节点的内容

"分裂，重分，中值往上"


### B+树删除

1. 找到叶子节点L
2. 移除之
	a. 如果移除后叶子节点仍是half-full的，结束
	b. 否则 **从姐妹节点中获取数据** ，调整至half-full
	c. 如果无法都包含half-full，与姐妹节点合并
3. 如果发生了合并，需要调整父节点的内容
4. 合并开销大，也可以考虑使用其他机制，而是不立即合并


### **B+树设计**

#### 节点大小

速度越慢，节点越大。 **IO随机访问慢，那一次尽可能多地读取连续的有效数据(一个node)**

当然具体情况具体分析

> What if Persistent Memory?


#### 合并阀值

合并开销大，许多DBMS在非half-full时也不触发合并。

可以删除时只做标记，也可以定期重建B+树


#### 变长key

对于变长的key该如何保存：

- 指针
	* 没利用局部性，慢
- 变长节点
	* 节点长度不定，索引值有待考虑
	* 内存管理，不能利用好page size
- Padding
	* padding到最大值，浪费空间
- **Key Map/Indirection**
	* **一个节点中** 保存有序key map来索引k-v，key map知道对应数据位置，使用offset索引
	* **优化** ：因为key map是有序的，所以可以在key map里保存一部分key用来初步过滤(二分查找)，而不是每次通过索引再去查找


#### 重复索引

- 保存多次
- 一key对多value(key是相同的)


#### Intra-Node Search

- 线性扫描
- 二分查找(有序)
- **Interpolation**

Interpolation: 我们知道/精心设计key的布局，那么就可以这些内容优化：e.g.

```
|---|---|-----|---|---|---|----|
| 4 | 5 | nil | 7 | 8 | 9 | 10 |
|---|---|-----|---|---|---|----|

要找8，就可以计算出offset = 7 - (10-8) = 5
```

#### 其他优化

- 前缀压缩
	* 因为是有序存储的，相邻key会比较相似，那么可以只保存一个公共前缀和各个不同后缀
	* "靠下的中间节点"
- 后缀截断
	* 中间节点只做索引，所以我们可以只保留足以区分的最小前缀，而去掉中间节点的后缀
	* "靠上的中间节点"
	* Q: 那怎么找到最小前缀呢?
- Bulk Insert
	* 建树时不是一个个插入，而是排序后自底向上构建，就不用频繁调整了
	* 利用这点，有序的叶子节点s是可以直接保存在磁盘的，然后Bulk Insert恢复
- Pointer Swizzling
	* 说是额外保存一个page id来索引节点来减少查页表，但是id找到指针后还是得查表
- **Pinned热点节点** ，如根节点/中间节点等经常被用到，那就固定在buffer中
- 联合查找索引失效问题
	* 所谓联合查找就是找多个属性的组合, e.g `(username, email)`
	* 因为根据索引b+树建树时可能会(具体实现相关)先根据第一个key排序，然后在相同的key内再排序
	* 所以`(username, email)`建立索引，`select username`索引有效，`select email`索引失效。公共左前缀原则
		+ 叶子节点顺序可能是`(1, 1) (1, 5) (2, 1) (2, 3)`，所以使用后缀是无法利用有序的性质的


## ch8 Table Indexs(II)

- 更多的技巧
	* 部分索引
	* Covering index

### 重复key问题

> 重复key会导致什么问题?

1. DBMS自动添加隐藏的`record id`和`key`一起作为unique id
2. 叶子节点"支持溢出"，链表或者某种方式把重复节点保存起来

### hints

#### 隐式索引

DBMD会自动为主码和unique的码建立索引


#### Partial Indexes

不用对该属性的所有内容做索引。可以只做部分索引 **如只对月份等于1的属性做索引** `month = 1`。之后搜索时，如果有月份为1的限制就可以使用索引。

```
create index idx_foo on foo where month = 1;
```

降低维护索引的开销。

小技巧：可以对每年，每月做部分索引。


#### Covering Indexes

索引用的key即所求的value，DBMS就不用再根据key去取value了。


#### Index Include Columns

索引内嵌额外信息(列)，这样索引查表时就可以少掉"查值"操作。道理类似Covering Indexes


#### Function/expression indexes

> an index dose not need to store keys in the some way that they appear in their base table

"多表索引"

具体DBMS支持不同


### 查表问题

- Trie Index
- Radix Tree

中间节点并不知道所查是否存在，每次都要从根遍历到叶子节点。"This means that you could have (at least) one buffer pool page miss per level in the tree just to find out a key do not exist". TODO 为何

所以有如下改进方法：


一种直观的方法是这样：用多少申请多少

```
 ┌─┐
 │H│
 └┬┘
  │
  │
 ┌▼┌─┐
 │i│a│
 └─┴┬┘
    │
   ┌▼┐
   │t│
   └─┘
```

另一种方法可以key span，每层保存所有可能出现的编码。如果key存在则有指针否则无。利用这种span思想的结构就是Radix Tree。

不过，上述方法存在一些不足之处，就每一层只判断一个字符，从而导致遍历深度问题，所以可以将"公共部分"/"不区分分区"压缩成一个节点。


```
   ┌─┐            ┌─┐
   │H│            │H│
   └┬┘            └┬┘
    │              │
    │              │
   ┌▼┌─┐          ┌▼┌────┐
   │i│e│   ==>    │i│ello│
   └─┴┬┘          └─┴────┘
      │
     ┌▼┐
     │l│
     └┬┘
      │
     ┌▼┐
     │l│
     └┬┘
      │
     ┌▼┐
     │o│
     └─┘
```

### 关键词搜索问题

思考下面的情形：你要找一篇文章中"你好"出现的位置/次数。这时索引就不那么好用了，因为索引都是"最右匹配的"，不会查询"子内容"。所以引入 **倒排查找** (Inverted Index，有时也称full-text search index)。

倒排索引就是建立`单词-目标内容`的map，直接用单词来索引文章内容。

上面虽说key的单词的map，但是需要具体情况具体分析，一般存在如下搜索情形：

- Phrase Searchs
	* 即简单的搜一个单词，从左到右匹配
- Proximity Searchs
	* 有范围的查询，如查找两个单词在n个单词之间出现次数
- Wildcard Searchs
	* 模糊查询，如正则表达式

针对以上情形，需要根据情况做出设计的决策：

- 要保存什么
	* 如可以保存单词出现的频率，位置或者其他meta-data等
- 什么时候更新
	* 维护一些辅助结构来一部分一部分的更新


## ch9 Multi-Threaded Index Concurrency Control

> 多线程索引并发控制
> 充分cpu多核性能和磁盘io的失速(stall)

- conclusion
	* 绞尽脑汁减少锁的范围
		+ crabbing
		+ 范围实在降低不了了就减低发生的频率
	* 我们发现热点节点root容易称为瓶颈节点，所以会考虑降低互斥在上层发生
	* 分裂/合并是并发性下降的根源
		+ 解决分裂：延迟分裂(delay parent update)
		+ 解决合并：延迟合并
		+ 降低不安全发生频率

**我们应该意识到: 合并和分裂这种不安全的情况是占少数的，或者我们可以让他尽量少发生** ，因为这些情况一旦发送，那就是大面积的锁和互斥。所以就有了如下方案：

- Crabibing："一步到位leaf锁，如不安全在老实重来"
- DBMS不立即触发 **合并** ，不安全频率再低
- **分裂** 时延迟parent的更新

> We focused on B+ Tree but the some high-level techniques are applicable to other data structure


### locks 和 latches

我们这里说的lock是 **针对事务** 的，在事务期间持有，必要时要回滚的。latch的 **针对临界区** 的，不需要考虑回滚。

| 区别项          | locks                | latches        |
|-----------------|----------------------|------------------|
| separate        | 事务间               | 线程间           |
| protect         | 数据库内容           | 内存数据         |
| during          | 整个事务             | 临界区           |
| deadlock探测    | 探测和解锁：超时终止 | 避免：编码的约束 |
| kept in(管理者) | lock manager         | 数据结构         |

latch模式：读写锁，即互斥锁和共享锁。


### 锁实现的几种方法

1. blocking OS mutex, e.g: `std::mutex`
	- not-scalable(core增长没用), 每次加锁解锁大概需要25ns
2. Test-and-Set Spin
	- 非常高效
	- not-scalable, **缓存不友好**
	- 自旋锁
3. Reader-Writer Latch，**读写锁(互斥锁)**
	- ...
	- 可以基于spin lock
	- 需要注意 **使用队列防止饥饿**
		* e.g.读锁在缓冲区，写锁发出申请，为了不让它饥饿，它和后面的R/W锁都如fifo队列


### Hash Table Latching

> "分桶"

1. 不会死锁，因为申请锁的顺序的相同的，没有交叉
2. 粒度可以是page, slot
3. 要resize可以对整个表加锁

方式：

1. Page Latch
	- 并发性低，但加/解锁开销也低
2. Slot Latch
	- 并发性高，但加/解锁开销也高(更频繁加解锁)


### B+ Tree Concurrency Control

> 实现多线程对B+树的操作

**问题驱动**：

1. 同时修改
2. split和merge

基本思想(步骤)：

1. parent申请锁
2. child申请锁
3. 如果child(包括一路的中间节点)是 **"安全的"** ，释放parent的锁，否则继续持有
	- 安全：不会导致分裂或合并
		* 即插入时没满，删除时还half-full

总结

- **尽可能减少锁的范围**
- 保证不会涉及到其他节点(split/merge)时(安全)，那么就跟访问单一共享资源是一样的
- assertion: 安全，修改值影响当前节点往下
- "持续持锁，安全解锁"


### **Latch Crabbing/Coupling: Better Latching Algo**

> 进一步降低锁的影响范围
> 思想：找到热点/瓶颈，那能不能从该瓶颈下手?

对于前面的策略，每次修改都要对root加互斥锁，此时root就称为了并发性的瓶颈。要是互斥能发生在靠近叶子的地方就好了。

**算法:**

先一路R锁，然后在目标叶子节点W锁，**如果不安全，则解锁然后用之前的方式重来**。

TODO 第一遍为什么需要R锁

- 体会
	* **充分考虑到分裂/合并(即不安全的情况)是少数** ，不必为了少数情况每次引入瓶颈
	* 第二遍会导致第一遍的R锁开销浪费了
	* 假设仅叶子节点修改
		+ 中间节点什么时候修改？
	* **有联系上之前说的DBMS可以不会立即触发合并**


### 另一种优化(瓶颈)的方向：down-top

目前所有线程都散top-down顺序加锁的，而又是树状结构，从而导致瓶颈。那么可不可以down-top的顺序加锁，"不先经过热点"?

- **利用sibling pointer，加锁方向有了更多可能**。
- 加锁方向多了，死锁就要多考虑了
	* 避免：申请不到锁要kill然后重试

但是我们定义的latch并不支持死锁检测(或通过判断是否持有锁)，所以只有**通过代码规范来解决死锁问题** : 

- 能经过sibling pointer的必须是"no-wait"的，如 **拿不锁到就返回失败**
- dbms要考虑到获取失败的情况，然后相应处理


### Delayed Parent Updates

> 合并/分裂的情况

每次叶子节点溢出我们都要更新：分裂的节点，新申请的节点和parent节点。就是说，因为分裂一会引入"大范围的锁"。

所以可以使用B-link-Tree(B+树变体)：在发送溢出时不立刻修改parent，而是标记该parent，然后用sibling指针将分裂的新节点插入叶子节点链表。如果下次修改再次访问到该parent的话才做分裂更新parent操作。

> 之所以可以link，是因为落到leaf后还要探测一下嘛，稍微修改一下探测策略(最多多探测下一个sibling)就可以了

```
                ┌─┬──┬─┬──┬─┬─────┐
                │ │5 │ │9 │ │M(14)│ 下次访问到再合并
                └┬┴──┴┬┴──┴┬┴─────┘
                 │    │    │
           ┌─────┘    └──┐ └───────┐
           │<5        <9 │         │>=9
           │             │         │
           │             │         │
           ▼             ▼         ▼
          ┌─┬─┬─┬──────►┌─┬───┬───►┌─┬───┐
 Leaf Node│1│3│ │       │6│7  │    │9│13 │      ┌─► ...
          └─┴─┴─┘◄──────┴─┴───┘◄───┴─┴───┤      │
                  Sibling                │  ┌──┬┘
                  Pointer                │  │14│
                                         └─►└──┘
```

严格B+树和实际需求(低不安全频率)的平衡。同理我们可以第三次，第四次再次访问时再合并。

> We focused on B+ Tree but the some high-level techniques are applicable to other data structure


## ch10 Sorting Aggregation

思考sql的大致流程: 找表，筛项，合并。而这个结果是很可能内存装不下的，所以我们的算法需要考虑外存的使用：**最大化连续访问**，利用外存顺序访问块的特点。

conclusion

- 一个基本思想是：**一次一batch(buffer/page)，尽量连续IO**
- buffer数量也可以充分利用，可以：
	* 多一份做预取
	* 多路归并
- **重新排序优先级甚至乱序io**
	* 一次一page  vs  每条记录都io
	* 可以测试一下
- 额外考虑不能fit in memory的情况

> 思考下面两种情况，哪个快
> 1. 重新内存排序，然后顺序io写回
> 2. index已排序，但是指向的value是乱序的，需要随机io取

### 排序

排序很有用，如

1. sql的集函数
2. sql的distinct, group by
3. bulk build的b+树


#### External Merge sort

归并就会想到分治，不过这回我们还要考虑到外存，要利用一次一page/buffer/block的思想。基本步骤：

1. Sorting：一block/page/buffer在内存排序，然后将排好的内容**写回一个文件**
2. Merging: 将这些个小文件合并到一个大文件

#### 两路归并排序(2-way externel merge sort)

每次将2 run合并到一个新run。

假设一共有B个buffer page:

1. pass 0: 每次读到B个page，然后在内存中排序
2. pass 1,2,3...: 递归合并2排好序的page到一个新page

- passse数: 1+logN: 1的第一次读入开销
- 总IO开销：2N x passes数：2表示读入写回

**两路归并的问题是：** 并不能充分利用B个page，只用了3page。


#### Double Buffering Optimization

思想：既然两路归并只用了3page，还剩很多可以用，**那就用一些来做预取吧**，从而减少等待IO的时间


#### General External Merge Sort

解设有B的buffer pagers，充分利用所有buffer资源做多路归并

1. pass0: 读入所有，在内存中排序，则需要N/B runs(一次读B page)
2. pass1,2,3...: 归并B-1 runs，**其中一个page做output**

- passes数：1 + $log_{B-1}[N/B]$
- 总IO开销：2N x passes数


### 利用B+树排序

如果有现成的B+树，也许可以用来加速排序：使用sibling指针遍历。

不过要考虑Clustered和Unclustered的B+树情况。

> B+树根据index排序
> Clustered: value是有序存储在一起的
> Unclustered: value是无序存储的，只是index有序可以指针找到


- 聚簇情况
	* 因为values存储的物理地址是有序的，io是顺序的，所以加速排序是没问题的
- 非聚簇情况
	* values存储的物理地址是乱序的，**每条记录可能导致读入新页，导致反复污染缓存池，io开销** ，还不如一次一page重排呢
	* 所以一般不会用非聚簇B+树来加速排序


### Aggregations

> 集函数如何妥善处理

两种聚合方式

- 排序
- hash


#### Sorting

开销比hash大的，**我们并不需要有序** ，只需要它的聚合功能。

> 去掉一些无用的特性，没准就优化了，所以hash yes

#### External Hashing Aggregate

> distinct: 去重
> group by: 集函数计算

- conclusion
	* 考虑buffer有限的情况

一般可以概括为两步：

1. Partition: 得到中间结果
	- 同样的, 一次一page，尽量连续IO
	- page满则写回disk
	- B-1 buffer做partition，1 buffer做input
2. Rehash: 中间结果再distinct, group by, 集函数等得到最终结果
	- 再读到内存，哈希分桶保存到**临时哈希表** ，然后计算最终结果(集函数, distinct等)i
	- 临时哈希表可以保存一些额外信息：如`groupKey-(cnt,sum)`等


## ch11 Join Algorithms

> 如何做连接(join)操作，制造一一对应关系

assertion:

- 小表叫"outer table"
- 大表叫"inner table"
- **我们总是让小表在左** ，然后用大表去"接"
	* 后面将会知道为什么: 遍历次数更少
	* **妙妙妙**

- conclusion
	* "原样复制"或Record Id
	* 一般连接都是等值连接，这等值就意味着也许某种"局部性"
	* IO开销分析别忘了可以利用page为单位
	* **为何outer talbe要小表**
	* 尤其考虑"IO复杂度"
	* 小心虚拟内存，不可控的随机换入出
	* DBMS要存在一定智能需要知道什么情况用什么策略
		+ e.g. 知道size小，那直接静态hash，建表探测开销都小
	* 时候sort join: 
		+ 没有唯一标识的数据
		+ 必须排序的情况(order by)

| Algorithm          | IO Cost      | Example  |
|--------------------|--------------|----------|
| Simple Nested Loop | M + (m x N)  | 1.3h     |
| Block Nested Loop  | M + M x N    | 50s      |
| Index Nested Loop  | M + M x C    | Variable |
| Sort-Merge         | M + N + sort | 0.59s    |
| Hash               | 3(M + N)     | 0.45     |


### 基本操作

- 扫描R表的元组r，扫描S表的元组s
- 当r和s达成某种条件是连接，输出到新表
- 新表的内容可以具体情况具体分析，如基于列存储的情况
	* 可以"原样拷贝"
		+ 好处是不用"指针找具体数据"
	* 也可以只 **记录record id** (相当于指针嘛)
		+ 新表会更小
		+ 但是查询数据要多一层用record id索引的工作
		+ **常用于基于列存储的情况(column store)**
		+ 也叫作 **late materialization**


### IO开销分析assertion

后面都假设：

- R表有M个page，m个元组(tuple)
- S表有N个page，n个元组(tuple)
- 并且我们计算开销只考虑IO开销，忽略部分计算开销
	* 所以可以page为单位的IO情况

### Nested Loop

> 多重循环

- Nested Loop 系算法
	* 小表"在外"
	* 多用buffer
	* 扫描/索引


#### Nested Loop Join: 双循环

```
foreach tuple r in R:
	foreach tuple s in S:
		emi, if r and s match
```

简单的双循环，需要 **注意的是**

- "outer"和"inner"
- 计算page为单位

所以总开销是：M + (m x N)

M是刚开始读入表的开销

- 一页多个元组
- R的m个元组都要遍历S的所有N页

**如果outer是大表** : 开销为：N + (n x M)

显然n的变化率要比M的变化率大。**这就是为什么outer要小表**

> abs: 循环次数外表定
>
> 外表的一项要和右表的每一个page的内容匹配, 所以左表决定了右表的遍历次数，希望遍历少点

e.g.: M = 1000, m = 100,000, N = 500, n = 40,000, 0.1ms/IO

Time = (1000 + (100000 x 500)) x 0.1 ms = 1.3h


#### Block Nested Loop Join

别忘了我们可以**一次一page**

```
foreach block(page) Br in R:
	foreach block(page) Bs in S:
		foreach tuple r in Br:
			foreach tuple s in Bs:
				emi, if r and s match
```

Cost: M + M x N

比较直观。

**用上更多的buffer**

```
foreach B-2 block(page) Br in R:
	foreach block(page) Bs in S:
		foreach tuple r in Br:
			foreach tuple s in Bs:
				emi, if r and s match
```

结果表用2 Block，一个存R输出，一个存S输出

Cost: M + (M/(B-2)) x N

如果B-2 > M，即内存能装下整个表。但一般数据TB, PB级

Cost: M + N

双循环出来什么问题? 从头到尾遍历


#### Index Nested Loop Join

用上Index加锁搜索匹配的过程。

```
foreach tuple r in R:
	foreach tuple s in Index(ri = si):
		emi, if r and s match
```

如果索引的开销是常数C的话，Cose: M + m x C


### Sort-Merge Join

1. 两表目标key排序
	- 之前的外部排序算法
2. "双游标遍历"，匹配拿出

有种"只需一次遍历"的感觉，因为有序后，内外表游标只会增或不变。**需要注意回溯的情况**:

```
	1 	1
 -> 2 	2 <= lastPosition
	2 	2
	3	2 <-
```

两个相同的元素会导致内表游标回溯到"lastPosition"，所以 **在有大量重复key的情况下的不利的**

开销：

- 左表排序开销(多buffer外排): 2M x (log M/log B)
	* 2M一读一写，每轮M/B次
- 右表排序开销: 2N x (log N/log B)
- 合并开销：M + N 回溯开销约掉了，相当于两个表正好走一遍
- 总开销：合并开销 + 排序开销

再考虑最坏的情况：所有key都是重复的，疯狂回溯。Cost：M x N + (sort cost)


### **Hash Join**

考虑到我们一般是做等值连接，不妨利用等值这个特性：等值hash结果相同。从而完成聚合操作。聚合完成后**虽然不同key哈希值可能相同，但是查找范围已经减小了，并且我们可多次哈希让范围更小**

整个过程抽象来看就是：建表 + 探测

#### Basic Hash Join Algorithm

1. 建表(Build)
	- 根据外表建立哈希表，相当于一个map
2. 探测(Probe)
	- 内表再用同样的哈希函数去这个map找，如果匹配就输出

所建哈希表的内容具体而定，不过用于判断匹配的key是必须的，当然还可以有一些其他属于，如用来判断">="等

同合并问题，哈希表可以存"原样拷贝"或所以

- Full Tuple
	* 不用再查表
	* 需要大量空间
- Tuple Identifier(record id)
	* 适用于基于列存储的
	* 适用于join selectivity低的情况(即不能一层层上去都是rid吧，太多索引了)


#### **布隆过滤器优化探测过程**

> 布隆过滤器常用于判断重复，哈希到bitmap所以空间小，内存友好
>
> 存在一定的误判率，可以使用多个哈希函数，所有哈希值都存在时才说明存在
>
> 可能会误判为存在，但不存在一定不会误判

建表的同时再建一个布隆过滤器:

1. 探测时**先用布隆过滤器过滤一遍**
2. 如果过滤结果存在，那么是"可能存在"，再用原始方法查表
3. 如果过滤结果不存在，那一定不存在，可以跳过

用布隆过滤器之所以能够加速的因为，布隆过滤器的bitmap，能够容纳在内存中，从而降低哈希的IO开销。


#### Grace Hash Join

上面的情况我们并没有考虑大数据内存不够用的问题，虽然我们的虚拟内存可以表现得"装得下"，但是这背后是**内存管理器大量的随机换入换出**

所以我们要**细分哈希表** ，每次尽可能局部性。

改：

- 建表(Partitioning)
	* 两个表都建哈希表，将大表分解成几个分区
- 探测
	* 每次在相应分区找就可以了

```
foreach tuple r in bucket(R,i)
	foreach tuple s in bucket(S,i)
		emit, if match(r, s)
```

"一次一page"，内存就够用了，就不用乱换出了

**如果一次hash还是太大那就再来几次**，不过需要额外机制记录谁哈希了几次的都是什么哈希函数。总之就是一一对应的关系要有。

IO开销：

- 建表: 2(M+N)，两个表都要一读一写
- 探测: M + N，一次一page比对
- 共3(M+N)


## ch12 Query Execution

> 理论基础就绪，如何编程，什么样的框架？

- conclusion
	* again，"局部性"，"隐式局部性": batch大小与硬件
	* 一次所有OLTP，一次一batch OLAP
	* 优化顺序查找：zone map，later
	* **如何正确使用多索引**: bitmap scan
	* **非聚簇索引的情况怎么优化** Index Scan选出key后，Page Sorting


### Processing Model

Processing Model: 如何获取数据，如何执行query plan，根据不同workload选择

- Iterator Model
- Materialization Model
- Vector / Batch Model


#### Iterrator Model

> 也称Volcano Model和Pipeline Model

> 类似迭代器

- 每个操作都实现一个`Next()`方法("抽象的下一步")
- 每次`Next()`调用**只返回一个元组**或空
- 递归(query plan)向下执行，所以一些操作会阻塞知道子操作执行完成(dfs)
- 这个方法的输出比较好控制
	* 一次就一个目标tuple，不用过多处理


#### Materialization Model

上面一次返回一个tuple，那这个就是**一次返回所有tuple**，多用点中间文件呗(那么显然接下来会用到一次一page)。

materialization指的是操作**将结果(所有tuple)materializes成一个结果**

- 当然DBMS可以设置参数限制输出的大小，防止过多tuple
- 输出就可以是所有属性(column)的tuple或执行列的所有tuple
- "大中间文件"
- **适用于数据量小的场景**，e.g. OLTP，因为OLTP访问的数据量小，输出所有tuple是ok的


#### Vectorization Model

- **Vectorization指的是cpu的向量指令**

前面要么一次一tuple，要么一次所有tuple，那么有没有折中的方案呢？那就**一次以batch吧**

Vectorization Model中每个操作产生的tuple的可配置的，每次取一batch。**batch的大小可以根据硬件资源设置，page size， cpu cache， 向量指令大小** !!!

- 适用于OLAP
	* OLAP数据量大，那就分治，一次一batch
	* OLAP一次一batch，执行的query数就比一次一tuple少了
	* 激发硬件性能(向量指令)，完成"analysis"工作


### Access Methods

如何访问具体存储的数据

- Sequential Scan
- Index Scan
- **Multi-Index / "Bitmap" Scan**


#### Sequential Scan

那就是双循环(tuples in pages)遍历呗，维护一个内部游标记录遍历到哪了。

但是这么简单双循环太不好了，我们可以采用如下优化：

- Zone Maps
- Late Materialization
- Head Clustering


##### Zone Maps

> Map as guide

Zone Maps就是在原始数据的基础上加**维护一些额外的信息**。**如对于一列，可以维护一个zone map，记录该列的最大，最小，平均值等**

DBMS查表时先通过Zone Maps过滤一下。

适用于OLAP，读多的场景。不适用与OLTP，OLTP频繁写让zone map维护成本高。


##### Later Materialization

> 之前说道的，"懒record id"思想

说的是执行的时候不用"每层"都生成临时文件，可以用挑出需要的record id，在最后需要的时候再拿到数据合成。


#### Index Scan

同join algorithm里介绍。


#### **Multi-Index Scan / Bitmap Scan**

ok，那么现在我们维护了多个index该如何用好这些索引呢？

```
where count > 30 and numbers > 100;
```

我们为什么会提出这个问题，是因为每个index维护在不同的B+树中，你用一个index选出后，这些数据在第二个index中要怎么用呢？比如我们可以在第一个index中找到后用sibling pointer一下子就扫描出来了。但是选出的数据在第二个index的B+树中可能是非常分散的，每个数据都搜一遍吗？第二个index不就没用了？

**这就是为什么叫Bitmap Scan**，体现了算法的精髓

我们可以:

- 同时**单独使用两个(多个)index**
- 使用相同的哈希函数，将匹配的record id哈希到一个"selected" bitmap中
- **将多个索引得到的bitmap做并/交/差**就得到我们最终需要的数据的record id
- "每个index都可以单独执行"(并发哦)


### **Index Scan Page Sorting**

我们再来看看那在非聚簇索引的情况下要怎么做。

我们说非聚簇所以key虽然是有序的，但是他们对应的value的物理位置(page id)是乱序的。一次有序key的扫描可能的结果是乱序的value访问，频繁切换buffer。所以非常低效。

那么可以这样

- 第一次扫描我们只用key(毕竟索引)，挑出所有需要的key
- 然后我们对这些数据，**根据page id排序**
- 最后在扫描获取结果

这样就避免了缓存池的污染


### Expression Evaluation

DBMS将`wher a = b`等操作组织成表达式树的方式来执行。(编译原理，通用，有优先级)

这么虽然通用，但是墨守陈规很慢，那么我们可以参考"编译优化"，将一些场景优化掉，如`where 1 = 1`，减少执行次数。


## ch13 Query Execution II: Parallel Execution

> 如何使用设计并发程序
>
> 为何要并行: 高吞吐量，低延迟，更少的钱做更多的事

- conclusion
	* 并发与局部性问题
		+ 污染，设计上小心顾此失彼
	* **并发并不能解决硬盘瓶颈问题**
		+ 甚至污染
	* thread其实也是某种bypass OS
		+ 很大定制空间，如定制DBMS调度
	* Intra, Inter parallelism
	* **水平扩展 + 垂直扩展**，即所谓的Bushy parallelism
		+ 水平，subset
		+ 垂直，pipeline
	* **IO parallelism**
		+ 分区(这不是分布式嘛)
		+ 行分列分

并发会带来cache污染，buffer污染

**很难做好做对**，需要考虑到方方面面，非常多细节: 

- 协作开销(并发任务)
- 调度方法(DBMS > OS)
- 并发问题(污染)
- 资源竞争(多核，并发，cycle增加)


### 并行 vs 分布式

- 并行
	* 比较可靠
	* 物理临近，通信快
- 分布式
	* 比较脆弱
	* 通信成本高


### Process Model

我们DBMS的worker优先级可以给得很高


#### Process Per Worker

每一个worker都是一个单独的进程。**历史原因**，因为当时还没有标志是多线程(thread)接口，为了可移植性。

- 依赖系统的调度
- 使用共享内存来通信
- 隔离性强: 一个进程crash不会影响其他


#### Process Pool

每个任务到来都会从多个可用worker中(进程池)分配一个执行。

- 同上
- **对cpu缓存不友好**
	* 污染cache

**并发也别忘了局部性**，设计上的顾此失彼。


#### Thread Per Worker

每个worker都是一个线程

- 依赖DBMS调度: **很大定制空间**
- 可以使用单独的dispatch线程
- 单个线程出问题，整个进程遭殃

优点就是：

- **更小上下文切换开销**(icache flush)
- 不需要管理共享内存


### 调度

关于调度我们应该考虑什么问题？

- 使用正确的worker
- **哪个cpu核，几个核**
- 如何处理输出

> thread某种意义上也是bypass OS了，从而可以具体定制


### Inter- vs. Intra-Query Parallelism

- Inter，表示query间并行
	* 提升整体性能
- Intra，表示一个execution中，子操作的并行
	* 单个执行更快


#### Inter-Query Parallelism

> 提升整体性能

指令间的并行。

如果是只读的操作间并行，ok cool，不需要多少通信。

比较tricky的是"写操作"的并行，hard to do correctly，16章。**最精彩的部分**


#### Intra-Query Parallelism

> 提升单个指令性能

指令内，子操作间的并行。

- 基本模型是生产者消费者模型
- 常用套路是：一个中心结构用于沟通并行或创建多个分区独立并行


#### Parallel Grace Hash Join

回忆Graec Hash Join。

```
                      bucket
                       │
                  join ▼
 ┌────┐  ┌───► ┌───┐ ┌───┐◄─────┐  ┌────┐
 │    │  │     └───┘ └───┘      │  │    │
 │    │  │                      │  │    │
 │    │  │     ┌───┐ ┌───┐      │  │    │
 │    │ h1 ──► └───┘ └───┘◄─────h1 │    │
 │    │  │                      │  │    │
 │    │  │     ┌───┐ ┌───┐      │  │    │
 └────┘  └───► └───┘ └───┘ ◄────┘  └────┘

```

- 两个建表操作可以并行
- 桶的join之间可以并行


### Exchange Operator

> "抽象"

- Type1 Gather: 合并
	* 合并workers产生的结果
- Type2 Repartition: 分类
	* 多个input分类
	* "mapreduce"
- Type3 Distribute: 分发
	* 一个input分发给多个worker

**一个Intra-Operator Parallelism的例子**

```

             ┌──────────┐
             │ exchange │
             └──────────┘
             ▲   ▲   ▲   ▲
             │   │   │   │
             σ   σ   σ   σ  
            ┌─┐ ┌─┐ ┌─┐ ┌─┐
            │ │ │ │ │ │ │ │
            └─┘ └─┘ └─┘ └─┘
             ▲   ▲   ▲   ▲
             │   │   └┐  │
             └───┼──┐ │  │
                 │  │ │  │
              ┌──┘  │ │  │
              │     │ │  │
            ┌─┴─────┴─┴──┤
  attibutes:│            │
            └────────────┘
```



### Intra-Operator Parallelism

- 水平扩展
	* 将一个操作分成多份，作用到不同subset中
	* 即不同子集间可以并行
- 垂直扩展
	* 流水线

水平扩展加速了一个操作的执行，不过仍存在一个操作要等待上一个操作执行完成，所以又考虑垂直扩展，流水线。

TODO 总结流水线思想

- 尽量让每一个worker都别停下来


**exchange抽象** : DBMS引入一个exchange操作来"协调"(合并，分类，分发)，类似于一个假的`Next()`下图也解释了为什么叫"水平"和"垂直"


### Bushy Parallelism

水平扩展，垂直扩展相结合。

```
        π
        ▲
        │
   ┌─►  ⋈   ◄─┐
   │   join   │
   σ          σ
   A          B
```

- 我们在"select"(`σ`)的时候可以使用intra parallelism，加速选出操作
- 然后join的地方建哈比表aggregate，可以一个bucket一个worker来并行


### IO Parallelism

并发虽好，但是硬盘问题仍然是一个大挑战。并发并不能解决硬盘瓶颈问题，甚至还会污染缓存。


#### Multi-Disk Parallelism

- 如可以将文件保存到多个Disk中，各个disk的IO相互相互独立
- 可以**根据实际情况使用不同的RAID**


#### Database Partitioning

- 基于"MAP": DBMS应该可以指定数据库使用的磁盘
	* buffer pool manager里修改一下page的映射就可以了
- 也可以基于文件系统
	* 根据不同目录划分
	* "mount"

其中的一个难点的"日志文件"的共享问题。其实已经有点分布式的味道了。


### 分区方法

可以使用逻辑分区来划分不同的物理。这样来实现对应用透明。

但是DBMS应该也要有点感知力。如在分布式数据库中，DBMS应该可以就近执行。

- Vertical Partitioning
	* 一个表的属性可以分开存储
	* 所以每个独立的存储都需要存储"行信息"
	* 适用于**根据列优化的情景**
- Horizontal Partitioning
	* 基于某种**"根据行聚类"**的方法，如哈希，range，predicate(谓词)

## 贯穿始终

- what if 用上所有能用的buffer会怎样
- 管好size尺度，别让有机会乱换出

## 秒不可言


- Crabbing
- 先布隆过滤器预判
- 一次一page
	* 一次一个 no，一次所有no, 一次一page yes
- thread不也是某种bypass吗
