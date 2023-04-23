---
title: 锁锁速查
author: 66RING
date: 2023-03-25
tags: 
- OS
- misc
mathjax: true
---

# Cheat sheet

> 枚举各种情况

- 普通自旋锁
    * 缓存与不可扩展
- MCS锁（排队锁）
    * 可扩展但开销更高
- QSpinLock
    * spinlock + MCS lock
- 以上都没考虑**numa的影响**
- Delegation Lock（代理锁）
    * 同类临界区操作都发送同一个NUMA中执行
    * 解决了NUMA带来的不同性能问题
- 以上都没考虑**异构核的影响**
- LibASL可扩展锁（乱序重排锁）
    * 不超时的情况下，大核可以插队优先
- 以上是通用锁的情况，还有读锁和写锁
- RWLock
    * 需要一个锁来修改和访问标记
- RCU
- PRWLock: Passive Read-Write lock
    * 使用版本号来判断冲突


## PRWLock

```c
// 写者
lock(write_lock)
global_ver++
// 等所有读者有就绪
for (i = 0; i < reader_cnt; i++)
	while (local_ver[i] != global_ver);
// 也可以主动更新读者的版本号

// 读者
// 更新版本号
while (is_locked(write_lock))
    local_ver[rid] = global_ver
```


