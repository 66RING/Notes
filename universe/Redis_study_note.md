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



