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
| <++>                     | <++>                     | <++> |
| <++>                     | <++>                     | <++> |


### 数
| command               | desc      | T(n) |
|-----------------------|-----------|------|
| incr key              | 自增1     | O(1) |
| decr key              | 自减1     | O(1) |
| incrby key k          | 自增k     | O(1) |
| decrby key k          | 自减k     | O(1) |
| incrbyfloat key float | 自增float | O(1) |
| <++>                  | <++>      | <++> |
| <++>                  | <++>      | <++> |
| <++>                  | <++>      | <++> |
| <++>                  | <++>      | <++> |
| <++>                  | <++>      | <++> |

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
| <++>                | <++>                         | <++> |
| <++>                | <++>                         | <++> |

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
| <++>                                    | <++>                                                    | <++>   |
| <++>                                    | <++>                                                    | <++>   |
| <++>                                    | <++>                                                    | <++>   |
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
| <++>                                | <++>                     | <++> |

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
| <++>                                | <++>                                 | <++>           |
| <++>                                | <++>                                 | <++>           |
| <++>                                | <++>                                 | <++>           |
