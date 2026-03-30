---
title: Frangipani - A Scalable Distributed File System
author: 66RING
date: 2022-06-30
tags: 
- paper
- distributed
mathjax: true
---

# Abstract

TODO: 加总结性的关键词

将许多磁盘管理成一个存储池

- 可扩展性
- 两层结构
	* 上层Frangipani服务
		+ 多开以提供扩展性
	* 底层Petal提供虚拟磁盘 + 分布式锁

## 正文

### System Structure

- 多机器运行Patel提供存储池, 大块的虚拟磁盘
- Frangipani运行为操作系统文件系统模块, 有fsync, sync语义(即缓存和写回)
- log保存在Patel中, 任意Frangipani宕机其他仍可恢复
- "以Patel为中心", Frangipani不直接与peer，而是与Patel通信和分布式锁


### TODO

- scalability
	* per-file lock尔
- 缓存一致性协议
- 死锁避免
