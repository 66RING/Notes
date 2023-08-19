---
title: Redis赏析
author: 66RING
date: 2023-04-05
tags: 
- misc
mathjax: true
---

# Abstract

主要记录Redis整体架构及其巧妙的设计。

- 主要结构体
    * sds(simple dynamic string)
    * dict
    * set: TODO zset
    * list
    * 其他: zmalloc TODO:
- 主要lib(抽象)
    * redis object
    * ae

- 主要亮点
    * epoll实现的异步库
    * 渐进式rehash
    * 流式read, write
    * redis

# Redis

## sds

> SDS(Simle Dynamic String), 特点二进制安全，动态扩容。

redis要对`char*`字符串进行改进。感觉有几点原因：1. 要通过网络传输的数据往往需要记录他们的传输长度信息。2. 传统的`char*`以`\0`标识结尾，但对于其他非string类型的数据, `\0`可能是一个有效数据，所以需要使用其他字段标识结尾，即所谓二进制安全。

主要就是额外申请一段空间作为header用于记录元数据，然后根据"协议"只有sds的api会操作这个header，header对用户完全透明。内存布局如下:

```
| sds header | data | \0 |
```

申请空间是连带header的空间一并申请, 但只用于sds库内部的操作, "协议"规定的如此的内存布局，就可以通过`ptr-sizeof(struct sdshdr)`来获取完成的header结构方便后面操作。为了兼容`char*`还会多申请一byte的空间储存`\0`。

```c
typedef char *sds;

// 从不使用sdshdr
struct sdshdr {
    long len;  // 字符串实际长度
    long free; // buf剩余可用空间
    char buf[]; // 大小0
};
```

所谓动态扩容其实就是根据元数据判断容量的情况自动申请新的空间, 然后返回新指针。

总结一下就有这些好处/八股/为什么不用c原本的字符串

1. 精妙的内存布局设计`buf[]`不占空间，但能指示一块连续的内存
2. O(1)的长度获取
3. 二进制安全, 不用依赖`\0`判断结束，但是为了兼容性也会自动添加`\0`
4. 预分配，自动扩容等策略保证实时性能


## list

### 双向list

处理基本的双向链表操作意外还考虑了对象模型和迭代器。

所谓对象模型就是如何让"面向过程的C语言"做更多面向对象的事。答案就是struct怎么规划。思路其实也很简单，value直接存储为`void*`, 因为使用者一定知道自己存的是什么，再自行做类型转换即可。然后为了应付`void*`存储复杂结构的情况，`struct list`中还会存储一些回调函数来复制。

```c
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

迭代器抽象本质就是1. 另起炉灶，不影响原变量。2. 对外暴露统一的接口，即迭代器接口。


### TODO: 其他list? ziplist?

## dict

> 渐进式rehash和"懒"超时检测

哈希表的扩容操作是开销的比较大的，而redis又是单线程, 所以要想办法妥善处理：渐进式rehash。

redis的dict实现使用的是拉链法, 所以渐进式rehash也比较简单。保留两个哈希表，一个新表一个旧表，使用`rehashidx`记录rehash到哪个桶了，如果哈希到的桶小于rehashidx就操作新表，否则就操作旧表。当所有迁移完毕就可以新旧互换开始新一轮了。解决单线程问题的核心思路就是rehash一点(默认迁移一个桶)，处理正事一点。

```c
typedef struct dictEntry {
    void *key;
	union {
	  void* val;
	  uint64_t u64;
	  int64_t s64;
	} v;;
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
  dictEntry **table;
  unsigned int size;
  unsigned int sizemask;
  unsigned int used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```

不过细节是魔鬼, 双表还有很多操作需要多加注意, 比如查找和删除是两个表都要查找。

至于过期的实现，也采用懒处理的方法。因为哈希表记录了过期的时间，当访问到元素时在判断是否过期，如果过期就删除即可。


## set

set即dict


## Redis Object

> redis object share everything

使用ref count做gc, 数据拷贝传递的是底层的redis object指针。

但是最后发现使用这种share everything的方式会导致大量的指针使用, 最终导致**指针跳来跳去影响cpu cache**, 从而降低性能。e.g. 你的访问数据会有如下调用:


## Server

## ae库


# 手写Redis

> 软件架构(整体) + 数据结构(细节)

- tips
    * 先看原始版本, e.g. v1.0, 不必第一个commit, 一个大版本整体性会更强
    * ⭐如何学习codebase: 基本概念, 流程(会用), 核心数据结构

redis结构, v1.0

- 事件库(ae)
    * ⭐, 可以学来做自己的server
- 网络库
- 内存库
- 数据结构: sds, 双向链表, dict
- utils

- **整体亮点**
    * 全异步, ae库
    * 渐进式rehash
    * 流式read, write
    * redis object share everything, 但是败给了cpu cache

## redis核心概念

> redis.c

- server核心
    * fd: 网络句柄
    * db: 多db(简化: 单db)
    * clients: 链接的client
    * eventloop: 事件循环
- client核心
    * fd
    * db
    * query: string buffer
    * reply: 一系列redis object
- redis DB
    * 本质就是哈希表, kv数据库: dict, map, etc...
    * expires map: 记录过期时间
    * ~~db id~~
- redis object ⭐
    * ptr: `void *`
    * type: 指向类型
    * refcount: 内存管理
- 核心流程
    1. 启动流程
    2. 请求处理流程: 怎么接收, 处理, 返回


## redis核心流程

> main()

- abs(本章亮点)
    * main
    * 事件注册的逻辑
    * epoll核心流程: 一批来一批去

1. 初始化
    - 初始化server
    - 监听端口
    - kv数据库
2. 主循环
3. ae事件注册
    - "accept event"
    - "client query event"
    - "write reply event"
4. 启动主循环

- 首先main注册事件监听端口: `acceptHandler`。一个链接产生一个配套fd, 那就可以依此创建一个client对象
- client对象处理: 非阻塞, 绑db等属性, **注册事件**
- readQueryFromClient
    1. read fd
    2. 处理请求processCommand, 如果是多个请求就again
- 处理请求
    1. lookup cmd: get, set... 查一个cmd table: type, 处理方法, 参数个数等
        - getCommand: 
    2. 执行命令
    3. ⭐写返回: addReply
        - 注册一个事件监听可写: sendReplyToClient
- 写回复sendReplyToClient
    * write, 然后删除事件

main注册accept事件(监听端口)。链接建立时(client到来), 为client的fd也注册事件监听, read event(readQueryFromClient)。client请求到来就调用命名处理。处理完毕后要返回给客户端, 调用一个write事件监听可写, 然后返回给客户端sendReplyToClient。


### ae库

> 就绪 -> 执行回调函数: **事件循环**

- 两种event的抽象
    * ⭐file event: everything is file. e.g. fd
    * time event
- event loop
    * 通过链表方式组织所有event
- API
    * `aeEventCreate`: 创event, 头插链表
    * `aeMain`: while !eventloop->stop
- 主要流程
    * wait返回所有就绪的epoll
    * procee处理这些epoll

- result
    * epoll API, TODO: 整理
        + 一个epollfd管理内部若干fd
        + epoll超时时间刚好又做计时器
    * 没等到file event就执行time event

TODO: 重新整理


## redis核心数据结构

> dict, string, redisObj, **多多考虑单线程下重操作要怎么妥善处理**

- abs(本章亮点)
    * 懒删除: expire
    * ⭐渐进式rehash(因为要照顾单线程的): 本质就是rehash一下, 执行kv一下
    * redis object: share everything
    * **最后redis object弃用**: 指针跳来跳去影响cache命中


- sds: redis的string库
    * TODO: 细节
- dict: kind of map
    * ⭐rehash: 怎么扩容又不影响吞吐时延
        + redis 2.6才有渐进式rehash
- redis object
    * 机制
    * 命令的生命周期: ref count, 怎么共享


### dict

- dictType: 哈希函数, keyCompare方法
- hashtable(`ht`): 简单的链式hash
    * **2.0中有两个**: 为了rehash
- expire: 过期的实现
    * ⭐单线程中是怎么干的⭐: 毕竟不可能放着主任务不管吧
    * 答案是用的时候再检查过期, 懒删除
        + `lookupKeyRead` -> `expireIfNeed`

- **⭐rehash**: 单线程场景下怎么妥善安排扩容
    * 如添加一条记录`dictAddRow`: 如果正在进行rehash则走rehash的方法, 否则直接诶添加
    * `dictRehash`: 一边rehash(默认走1步: 一个链表上的kv)一边操作kv: rehash一下, 操作kv一下尔
        1. 如果元表rehash完成了就free, 然后换表
        2. 使用rehashidx记录rehash到哪个桶了, 加一层while跳过本身就是空的桶
        3. 迁移该桶rehash + 置空, 更新新表旧表大小


e.g.: 创建一个dictType

```c
dictType dbDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictRedisObjectDestructor   /* val destructor */
};
```


### Redis Object

> redis object的存在可以减少拷贝的次数。命令行到来产生数据, 数据拷贝到dict中。查询时又要从dict中拷贝数据返回。通过所有数据都存储成robj可以大幅减少拷贝(但后面不用了)

- ref count做gc
    * count = 0时释放
    * 缺点: 循环依赖会泄露, 但redis的场景不存在这个问题
- 命令处理时就生成robj, 存储到`redisClient->argv`
- 以set为例
    1. processComand -> robj
    2. dictAdd robj存储
    3. incrRefCount robj引用计数++
    4. resetClient: 释放robj, 引用计数--
    5. 至此先加再减, 又回到了原来的dict中的一份

但是最后发现使用这种share everything的方式会导致大量的指针使用, 最终导致**指针跳来跳去影响cpu cache**, 从而降低性能。e.g. 你的访问数据会有如下调用:

```
key -> key_obj -> hash table -> robj -> sds_string
````

所以最后就直接存储具体数据了


### sds

> simple dynamic string

- abs
    * 内存布局
    * 动态扩容
    * sdsnew

sds直接操纵内存布局, 其内存布局如下:

```
| sds header | data | \0 |
```

对应结构就是下面这样, sds使用sdshdr申请空间, 返回时只返回指向buf部分的指针。需要访问header时再向前找: `s-sizeof(struct sdshdr)`

```c
typedef char *sds;

// 从不使用sdshdr
struct sdshdr {
    long len;  // 字符串实际长度
    long free; // buf剩余可用空间
    char buf[]; // 大小0
};
```

虽然有sdshdr这个结构但是从不直接使用, 因为我们要连续空间那么buf是个有实际大小的指针就说不过去。那么它如何实现动态扩容? 其实就是自动数据迁移后返回返回新指针(v1.0)。因为在1.0版本中还没有动态添加等操作, 只有cpy等操作遇到容量不够时自动分配，然后也是返回新指针。

- 好处/八股/为什么不用c的字符串
    1. O(1)的长度获取
    2. 二进制安全, 因为用的len字段表达互长度, 而不是`\0`
    3. 防止了缓冲区溢出, sds API自动扩容
        - 扩容逻辑, 小空间时(e.g. 小于1MB)直接翻倍, 大空间时每次加1MB
    4. 预分配

TODO:

### zset

TODO:

## code

- 基本数据结构: string, dict, zset, set, list
- 要在golang中使用系统调用


- ae库
- net库: 因为golang中的net库太高层了, 我们自己封装系统调用
- epoll
    * 设置一定的超时时间, 因为有time event要处理的
    * 每次收集一批file event和time event然后做process

... TODO


## redis协议

- RESP
    * 数据流式读取过程
- 以QueryFromClient为例

- abs(本节亮点)
    * TODO
    * ⭐流式read, write怎么处理⭐

### RESP协议

> 格式: 首字母 + 内容 + 换行\r\n(CRLF)

首字母有如下几种(redis2.6)

- `+`, 简单string
- `-`, 错误
- `:`, 整数, 如返回数字的情况
- `$`, 复杂string(string内可能还有换行)
    * e.g. `$6\r\nsome string\r\n`, `$`后跟string长度用\r\n表示长度结束
- `*`, 数组, 星号后接数组长度, 之后每个元素用\r\n隔开
    * e.g. `*2\r\n$3\r\nabc\r\n$3\r\nbar\r\n`

⭐难点在于流式处理, 如read一个长度3的数组, 但是只read了一半

TODO: queryBuf怎么处理。queryBuf动态增长, 新数据写到queryLen之后

processBuf

回复同理, 我们不能进行阻塞式地写, 所以会将reply保存到client中的一个list中, 可写时再用回调写回(SendReplyToClient)。那么问题就来说我们一次可以write的byte的可能有限，需要处理流式write。


## dict实现


## 最小化

- cmdTable
    * get
    * set
    * expire

测试

```
$ telnet <ip> <port>
set k1 v1
set k2 v2
expire k1 8
get k1
get k2
...
get k1
get k2
```

### 扩展

- 支持redis官方client
    * auth命令: 认证
    * command命令:  询问server支持什么命令
- godis client⭐, redis client设计也很好
- 其他数据结构
- 持久化, 主存
- cluster



## 个人项目上的应用

二进制数据的传输 = header + data


