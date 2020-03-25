---
title: Redis 学习笔记
date: 2020-3-18
---

# Redis学习笔记

## 基本认识

- Redis 是单线程的, 也就是说在处理不当会导致阻塞
    - 不要使用长命令, 如: keys *


### 特性

- 高速
    - 内存中进行的
- <++>
- <++>
- <++>
- <++>
- <++>
- <++>


## 通用命令

### get

| command             | desc                            | T(n) |
|---------------------|---------------------------------|------|
| keys [pattern]      | 根据通配符号检索key             | O(n) |
| get key             | 获取value                       | O(1) |
| mget key1 key2...   | 批量获取value                   | O(n) |
| getset key newvalue | set key newvalue并反会旧的value | O(1) |
| append key value    | 追加                            | O(1) |
| strlen key          | 长度                            | O(1) |


### set

| command                         | desc                                 | T(n) |
|---------------------------------|--------------------------------------|------|
| set key value                   | 设置 key value                       | O(1) |
| mset key1 value1 key2 value2... | 批量设置 key value                   | O(1) |
| setnx key value                 | 如果key不存在,设置 key value         | O(1) |
| set key value xx                | 如果key存在,设置 key value           | O(1) |
| dbsize                          | 计算key总数                          | O(1) |
| exists key                      | 判断存在                             | O(1) |
| del key [key ...]               | 删除                                 | O(1) |
| expire key seconds              | key在seconds秒后过期                 | O(1) |
| ttl key                         | 查看过期时间, -1没设置过期, -2已过期 | O(1) |
| persist key                     | 去掉过期时间                         | O(1) |
| type key                        | 返回类型                             | O(1) |


### 字符串

| command                  | desc                     | T(n) |
|--------------------------|--------------------------|------|
| getrange key start end   | 获取字符串指定下标所有值 | O(1) |
| setrange key index value | 设置指定下标对应值       | O(1) |


### 数

| command               | desc      | T(n) |
|-----------------------|-----------|------|
| incr key              | 自增1     | O(1) |
| decr key              | 自减1     | O(1) |
| incrby key k          | 自增k     | O(1) |
| decrby key k          | 自减k     | O(1) |
| incrbyfloat key float | 自增float | O(1) |


### 哈希

结构: key -> (field -> value)

hash的所有命令都是h开头的
| command             | desc                         | T(n) |
|---------------------|------------------------------|------|
| hget                | <++>                         | <++> |
| hset                | <++>                         | <++> |
| hdel                | <++>                         | <++> |
| hexists key field   | <++>                         | <++> |
| hlen key            | count field                  | <++> |
| hmget               | <++>                         | <++> |
| hmset               | <++>                         | <++> |
| hincrby key value k | incrby k                     | <++> |
| hgetall             | get all (field,value) by key | O(n) |
| hvals key           | get all values by key        | <++> |
| hkeys key           | get all fields by key        | <++> |


### 列表

结构: key -> [list]

list的所有命令都是l开头的
| command                                 | desc                                                    | T(n)   |
|-----------------------------------------|---------------------------------------------------------|--------|
| rpush key value1 value2 ...             | 从列表右端插入                                          | O(1-n) |
| lpush key value1 value2 ...             | 从列表左端插入                                          | O(1-n) |
| linsert key before/after value newvalue | 在指定的value前/后插入newvalue                          | O(n)   |
| lpop key                                | 从列表左边弹出                                          | O(1)   |
| lrem key count value                    | 从左边删除count个value, 删除重复元素, count<0从右边删除 | O(n)   |
| ltrim key start end                     | 裁剪出制定范围的元素                                    | O(n)   |
| lrange key start end(包含end)           | 获取指定范围的元素                                      | O(n)   |
| lindex key index                        | 索引取出                                                | O(n)   |
| llen key                                |                                                         | O(1)   |
| lset key index newvalue                 | 按照索引修改指                                          | <++>   |

### 无序集合

结构: key -> set
- 无需
- 无重复
- 支持集合间操作

set所有命令s开头
| command                             | desc                     | T(n) |
|-------------------------------------|--------------------------|------|
| sadd set element                    | insert element           | O(1) |
| srem set element                    | delete element           | O(1) |
| scard set                           | count element inside set | <++> |
| sismenber set                       | check if exists          | <++> |
| srandmember set count               | 随机取出count个          | <++> |
| spop set                            | 随机弹出1个              | <++> |
| smembers set                        | get all element          | <++> |
| sdiff set1 set2                     | 差集                     | <++> |
| sinter set1 set2                    | 交集                     | <++> |
| sunion set1 set2                    | 并集                     | <++> |
| sdiff/sinter/sunion + store destkey | 把结果保存到destkey中    | <++> |


### 有序集合

结构: key -> (score -> element)
- element不可重复
- 顺序由score定

有序set所有命令z开头
| command                             | desc                                 | T(n)           |
|-------------------------------------|--------------------------------------|----------------|
| zadd key score element              | <++>                                 | $O(\log n)$    |
| zrem key element                    | <++>                                 | <++>           |
| zscore key element                  | get score by element                 | O(1)           |
| zincrby key increScore element      | 给element增加指定分数                | <++>           |
| zcard key                           |                                      | 返回个数       |
| zrank                               | 获取排名                             | <++>           |
| zrange key start end [withscores]   | 获取范围withscores选项是是否打印分值 | $O(\log(n)+m)$ |
| zrangebyscore key minScore maxScore | <++>                                 | <++>           |
| zcount key minScore maxScore        | <++>                                 | <++>           |
| zremrangebyrank key start end       | <++>                                 | <++>           |
| zremrangebyscore key start end      | <++>                                 | <++>           |


### 慢查询

生命周期
``` sequence-diagrams
client->command queue: command
Note right of command queue: Execute one command
command queue-->client: result
```

慢查询发生在执行命令的过程, 如`keys *`就会发生慢查询, 通过配置慢查询来防止阻塞

| 配置                                     | 结果                                               |
|------------------------------------------|----------------------------------------------------|
| config set slowlog-max-len value         | 慢查询队列的最大长度为value                        |
| config set slowlog-log-slower-than value | 把慢于value微妙的命令放入慢查询队列, 一般设置1微妙 |
| slowlog get [n]                          | 获取慢查询队列                                     |
| slowlog len                              | 获取慢查询队列条数                                 |
| slowlog reset                            | 清空                                               |

### 流水线 Pipline

网络通信时间 = 网络时间+命令时间

因为redis很快, 通信时间大多数时候受限于网络时间, 而且如果让redis同时mget, mset或者n次set是不行的, 这意味着就需要多次请求, 就会耗费很多时间

流水线的作用就是把一批命令批量打包, 发送到服务端, 然后按顺序反回, 这样n次网络时间就缩短为1次了


### 发布订阅

``` sequence-diagrams
发布者publisher->频道redis server: 发布消息
频道channle-->订阅者subscribers: 订阅消息给订阅者1
频道channle-->订阅者subscribers: 订阅消息给订阅者2
频道channle-->订阅者subscribers: 订阅消息给订阅者3
Note right of 订阅者subscribers: 每个订阅了频道的人都会收到消息
```

| API                               | desc                               |
|-----------------------------------|------------------------------------|
| publish channel message           | 发送message到channel, 返回订阅者数 |
| subscribe [channel] #一个或多个   | 订阅                               |
| unsubscribe [channel] #一个或多个 | 取消订阅                           |

**消息队列** 类似发布订阅, 只是只有一个订阅者能收到, 类似强红包


### 位图Bitmap

redis中对位进行操作, 字符串其实就是字符数组嘛, 每个字符又是一段二进制编码

| API                               | desc                                                                          |
|-----------------------------------|-------------------------------------------------------------------------------|
| setbit key offset value           | 设置, value只能是0,1                                                          |
| getbit key offset                 |                                                                               |
| bitcount key [start end]          | 获取指定范围1的个数                                                           |
| btop op destkey key [key...]      | 将多个位图进行交并非异或等操作, 把结果保存到destkey中                         |
| bitpos key tagetBit [start] [end] | 计算位图指定范围(start, end单位是字节)第一个偏移量对应的值等于targetBit的位置 |


### HyperLogLog

用极小的空间完成独立用户的统计
- 有错误率
- 不能取单条数据

本质结构还是string 
| API                                      | desc                      |
|------------------------------------------|---------------------------|
| pfadd key element [element..]            | 向hyperloglog添加元素     |
| pfcount key [key..]                      | 计算hyperloglog的独立总数 |
| pfmerge destkey sourcekey [sourcekey...] | 合并多个hyperloglog       |


### 地理信息定位GEO

存储经纬度, 计算两地距离, 范围计算等

| API                                     | desc             |
|-----------------------------------------|------------------|
| geo key longitude latitude member [...] | 添加地理位置信息 |
| geopos key member [...]                 | 获取信息         |
| geodist member1 member2                 | 算距离           |
| georadius                               | 范围             |


## 持久化

由于Redis是将数据保存在内存中的, 所以需要持久化来异步的保存到磁盘上


### RDB

生成快照, 保存到硬盘中实现持久化

自动保存默认配置
``` redis
save 900 1    # 如果900秒内改变了1条内容就自动保存
save 300 10
save 60 10000
dbfilename dump.db  # 默认保存的文件
dir ./
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes  # 检验
```

| API    | desc                                                             | T(n) |
|--------|------------------------------------------------------------------|------|
| save   | 保存, 会造成阻塞                                                 | O(n) |
| bgsave | 异步保存, 通过子进程来生成RDB, 也会阻塞(发生在fork中), 但非常快 | O(n) |


### AOF

RDB的问题
- 耗时, 耗性能
    - 生成快照会把整个文件保存到硬盘
- 不可控, 丢失数据
    - 不论是自动保存还是手动保存, 都不可避免因宕机导致的数据丢失

AOF 类似写日志的形式保存每条redis命令, 恢复再根据这些命令恢复

AOF也不是直接把数据写道硬盘中, 那样很慢, 而是将数据写道缓冲区, 再根据策略写到硬盘

AOF的三种策略(默认)
- always
    - 每条命令都写到硬盘, I/O开销很大
- everysec
    - 每秒到写到硬盘, 有可能丢失1秒数据
- no
    - 操作系统决定什么时候写就什么时候写


#### AOF重写

如set同一个key多次只保留最后一次的内容, 不保留过期的命令等等...

AOF重写实现的两种方式
- bgrewriteaof
    - 类似bgsave, 在子进程中进行
- 使用重写配置
    - 配置
        - `auto-aof-rewrite-min-size`: AOF文件重写需要的尺寸
        - `auto-aof-rewrite-percentage`: AOF文件重写需要的增长率
    - 统计
        - `aof_current_size`: AOF当前尺寸(字节)
        - `aof_base_size`: AOF上次启动和重写的尺寸
        - 配合配置就可以在一定时机重写

可用AOF配置
``` redis
appendonly yes
appendfilename "appendonly-${port}.aof"
appendsync everysec
dir ./bigdiskpath
no-appendfsync-on-rewrite yes
auto-aof-rewite-percentage 100
auto-aof-rewite-min-size 64mb
```


### RDB和AOF选择

| 命令       | ROD      | AOF        |
|------------|----------|------------|
| 启动优先级 | 低       | 高         |
| 体积       | 小       | 大         |
| 恢复速度   | 快       | 慢         |
| 数据安全性 | 丢失数据 | 有策略据定 |
| 轻重       | 重       | 轻         |


## Redis复制原理

### 主从复制

单机运行redis可能面临许多风险, I/O瓶颈, 宕机风险, qbs瓶颈. 等等问题

使用redis的主从复制就能很方便是实现一个高可用的分布式数据库

- 一个master可以有多个slave
- 一个slave只能有一个master
- 数据流向是单向的, master到slave

redis中从节点slave相当于主节点master的备份, 当master中`set key value`后, 从节点也进行的同样的操作


### 实现方式

- 命令实现: 
    - `redis-6380> slaveof 127.0.0.1 6379`6380成为6379的从复制
    - 取消复制`slaveof no one`, 不成为任何人的从节点
- 修改配置文件
    - `slave of ip port`
    - `slave-read-only yes`, 从节点不做任何写的操作, 保证和主节点数据一样


### 全量复制

Redis的主从复制是异步操作, 也就是说同步过程中, 主节点可以发生改变, redis有机制能够保证同步过程发生的改变也同步到从节点中. 也就是全量复制

**全量复制过程**
- 从主节点复制, 生成rdb文件
- redis内部会记录复制期间主节点的变化
- 复制完成后比叫主从节点的偏移量来查看主节点是否有变化, 并将变化同步到从节点中

**全量复制开销**
- bgsave时间
- RDB文件网络传输时间
- 从节点清空时间
- 从节点加载RDB文件时间
- 可能的AOF重写时间


### 部分复制

如果全量复制期间发生网络抖动等原因导致复制到从节点数据丢失或不完整, 再次进行全量复制显然是不合理的, 因为前面看到全量复制开销很大. 这时就有了部分复制

在全量复制开始时, 主节点会进行复制缓冲区命令. 当网络抖动结束后, 从节点再次尝试连接主节点, 并将自己的偏移量offset和runid发送给主节点
如果从节点的偏移量和主节点的偏移量小于某值(缓冲区内), 就会发生部分复制, 从从节点的offset开始复制, 发送给从节点.
否则就全量复制.


### 故障处理

- slave故障
    - 使用别的slave暂时代替
- master故障
    - 让slave成为新的master


### 主从复制常见问题

- 读写分离
    - 用master写, 读的流量分摊到各个slave
    - 可能问题:
        - 复制数据延迟
        - 读到过期数据
        - 从节点故障
- 配置不一致
    - 如maxmamory不一致导致数据丢失
    - 如数据结构优化参数导致内存不一致
- 规避全量复制
    - 第一次全量复制不可避免
        - 小主节点, 或者低峰值的时候进行
    - 节点运行ID不匹配
        - 主节点重启(运行ID变化)
        - 故障转移
    - 复制积压缓冲区不足
        - 部分复制
        - 增大复制换缓冲区配置`rel_backlog_size`
- 复制风暴(主节点挂载了很多子节点, 当主节点宕机重启后, ID发生变化, 所有从节点都进行一次全量复制)
    - 单节点复制风暴
        - 分摊从节点到从节点的链式结构, 可以减轻主节点的压力, 当时结果复杂, 维护难度加大
    - 单机复制风暴, 单机器上都是主节点
        - 主节点分散多机器


## Redis Sentinel

Redis Sentinel是redis的一个高可用实现, 解决故障节点的检测和完美迁移的一个客户端


### Redis Sentinel框架

sentinel会对每个redis进行监控, 用户使用redis不再是直接使用redis, 而是通过sentinel间接使用redis.
当发生故障进行故障转移后, 客户端不再关心哪个redis成为了master, 只用关心sentinel告诉我们的结果. 

Redis Sentinel其实是个sentinel集合, sentinel实现故障转移的过程如下:
- 多个sentinel发现并确认master有问题
- 选举出一个sentinel作为领导
- 选出一个slave作为master
- 通知其余slave成为新master的slave
- 告知客户端主从变化
- 当老的master复活后, 使其成为新master的slave

Redis Sentinel可以监控多套master和slave, 使用master-name的配置作为标识和说明


### Redis Sentinel原理

Redis Sentinel内部会运行三个定时任务
- 每10秒每个sentinal对master和slave执行info
    - 发现slave节点
    - 确认主从关系
- 每2秒每个sentinel通过master节点的channel交换信息(发布订阅)
    - 通过 \_\_sentinel\_\_:hello频道交互
    - 交互节点的"看法"(失败判定)和自身信息
- 每1秒每个sentinel对其他sentinel和redis执行ping
    - 心跳检测, 失败判定


#### 主观下线和客观下线

每个sentinel对节点的判定可能不同(受网络影响等等), 所以一个节点的判定是不真实的, 根据单个节点进行的主观下线也是不合理的.
所以下线操作要根据多个节点投票来决定, 称之为客观下线.

`sentinel monitor <masterName> <ip> <port> <quorum>`quorum就是投票通过的判定, 多为奇数


#### 领导者选举

领导者选举使用raft算法

- 选举: 通过sentinel is-master-down-by-addr命令都希望成为领导者
    - 每个主管下线的sentinel节点向其他sentinel节点发送该命令, 希望将它设置为领导者
    - 收到命令的sentinel如果没有同意过其他sentinel节点发送的命令, 就会同意该请求, 否则拒绝
    - 如果当前sentinal节点发现自己的票数超过sentinel集合半数且超过quorum, 则它成为领导者
    - 如果此过程有多个sentinel节点成为领导者, 那么将等待一段时间再次进行选举


#### 故障转移(sentinel领导者完成)

- 从slave节点选出"合适的"节点作为新的master
- 对上面的slave节点执行`slaveof no one`让其成为master
- 向剩余的slave节点发送命令, 让它们成为新的master节点的slave, 复制规则和`parallel-syncs`参数有关
- 更新对原来master节点的配置为slave, 并保持这对其"关注", 当其恢复后命令它去复制新的master节点

**slave节点的选择** 
- 选择`slave-priority`(节点优先级)最高的slave节点, 如果存在则返回, 不存在进行下一步选择
    - 默认优先级的一样的, 可以根据机器配置高低手动配置
- 选择复制偏移量最大的slave节点(越大说明越接近主节点, 越完整), 不存在则进行下一步
- 选择runId最小的slave节点


### 安装

- 配置开启主节点
- 配置开启sentinel监控主节点(**sentinel是特殊的redis**)
- 实际应该用多机器
- 详细配置节点

配置开启主从节点

``` redis
port ${port}
daemonize yes
pidfile /path/to/pidfile.pid
logfile "logfile.log"
dir "/dir/to"

# 从节点
slaveof ip port
```

sentinel主要配置

``` redis
port ${port}
daemonize yes
dir "path/to/dir"
logfile "${port}.log"

sentinel monitor master-name ip port 2  # 自定一个mastername, 2表示当两个sentinel认为这个主节点出问题时, 故障转移
sentinel down-after-milliseconds master-name 30000  # 类似于ping的时间
sentinel parallel-syncs master-name 1  # 允许老slave同时对新master进行复制的个数
sentinel failover-timeout master-name 180000
```

启动Redis Sentinel
- `redis-sentinel [config file]`
- 或者`redis-server --sentinel [config file]`


### 客户端结合sentinel实现高可用

sentinel实现的高可用, 如果客户端没有实现高可用, 对sentinel的感知不高, 那也并不是高可用的
- 客户端遍历sentinel节点集合, 获取一个可用的sentinel节点
- 通过sentinal获取master节点`sentinel get-master-addr-by-name master-name`
- `role`或者`role replication`验证是否是真的master节点
- 当master节点发生变化, 通知客户端(发布订阅的模式)

各语言API有所不同


#### 读写分离

主节点挂掉后, sentinel服务可以方便的帮我们完成故障迁移. 如果slave节点挂掉后, 我们应该通过一定的手段让客户端迁移, 不让故障节点影响到客户端.

三个迁移判断:
- 切换主节点时(从节点晋升为主节点)
- 写换从节点时(主节点下降为从节点)
- 主观下线时

客户端配合sentinel实现起来还是比较复杂的, 后面的集群更适合进行这样的读写分离


### 主动下线

有时候设备更新等原因, 需要关闭/重启/迁移服务器等等. 运维人员可以手动下线服务器的节点. 
`sentinel failover <masterName>` 手动让主机点下线, 然后sentinel会让它故障转移.


### 节点上线

- 主节点: `sentinel failover <masterName>` 让重启的主节点重新成为主节点
- 从节点: `slaveof`即可, sentinel节点可以感知
- sentinel节点: 参考其他sentinel节点启动即可, 订阅发布


## Redis Cluster

### 呼唤集群

为什么需要呼唤
- 并发量: OPS
    - Redis官方数据每秒可以执行10万行, 但是如果业务要求100万/每秒呢
- 数据量
    - 一般的计算机内存16~256G, 但是如果业务要求500G呢
- 流量
    - 单机流量是千兆网卡, 但是业务要求万兆呢

分布式就是结局这些问题的好方法, 集群就相当于机柜. 规模化需求.


### 数据分布

全量分布 => 一定的分区规则 => 子集N

- 顺序分区
    - 数据分散度易倾斜
    - 键值业务相关
        - 如按日期的顺序分区
    - 可顺序访问
    - 不支持批量操作
- 哈希分区
    - 数据分散度高
    - 键值与业务无关
    - 无法顺序访问
    - 支持批量操作


#### 哈希分布

- 节点取余分区
    - 扩容时, 迁移率高
    - 使用多倍扩容可以优化
- 一致性哈希分区
    - 让节点分布在一个token环上(首尾相连的数组), 举行哈希运算后先后检索
    - 节点伸缩时, 只影响附近的节点, 迁移量较小
    - 伸缩时建议翻倍伸缩, 保证负载均衡
- 虚拟槽分区
    - Redis Cluster的分区方式, Redis Cluster有16384个节点, 开槽是对16384进行平均
    - 按一定的范围划分虚拟槽, 每个节点对应一个槽, 任取一个节点计算key哈希运算的结果, 如果结果是是自己管辖的范围, 则通过redis cluster的节点通信告知目标节点


### 集群搭建

Redis Cluster架构
- 节点
    - `cluster-enabled: yes`以集群模式启动节点
- meet
    - 有一个节点向其他节点发送"meet", 节点收到后回复
    - 内部机制让于同一个节点"meet"的节点们能互通
- 指派槽
    - 负载均衡
- 复制
    - 保证高可用


### 安装

#### 原生命令安装

- 配置开启节点
    - `port ${port}`
    - `daemonize yes`
    - `dir "/path/to/"`
    - `dbfilename "dump-${port}.rdb"`
    - `logfile "${port}.log"`
    - `cluster-enabled yes`  是cluser节点
    - `cluster-config-file nodes-${port}.conf` 为当前节点单独选择配置文件
    - `cluster-node-timeout 15000`
    - `cluster-require-full-coverage yes`  当集群中所有节点可以用时才提供服务, 默认是yes, 不符合高可用
- 节点握手: meet
    - `cluster meet ip port`
- 指派槽
    - `cluster addslots slot [slot...]`
- 主从关系分配实现高可用
    - `cluster replicate node-id` node-id是值集群节点的id, 不是runid


#### 使用Ruby安装脚本





