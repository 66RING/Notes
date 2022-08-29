---
title: Bitcask A Log-Structured Hash Table for Fast Key/Value Data
author: 66RING
date: 2022-08-29
tags: 
- database
- lsm tree
mathjax: true
---

# Abstract

Bitcask是一种基于哈希表的Log-Structured的KV存储结构。特点是**简单**, 且有足够好的速度和质量。

与LSM tree有点类似, 都利用了内存和外存配合, merge。

内存中的keydir哈希表使用文件和位置信息做数据索引(value = file + offset), 外存的log-structured做数据存储。

# Overview

每个bitcask实例都是一个文件夹, 内部有若干个日志文件用于记录数据和"backup"。操作系统每次只会打开一个文件, 该文件称为"active file"。active file大小到达阈值后将创建新active file。

- 读写模型:
	* bitcask中数据即日志, 以Append only的方式写入到active file中。而数据的索引则通过**内存中的keydir哈希表**获取到value所在的文件和位置
- 内存结构: keydir哈希表
	* 通过key所以value所谓的文件和文件中的位置
	* 初始化时从老版本到新版本扫描所有文件, 读取内容和位置信息建立内存中的哈希表
- 外存结构
	* append only, 对数据序列化
- 初始化
	* 初始化时从老版本到新版本扫描所有文件, 读取内容和位置信息建立内存中的哈希表
- 增删改查
	* 增改
		+ 只有一个文件可写, 那就是active file, 所以系统中只有一个writer
		+ append, 更新keydir
	* 删
		+ append"墓碑"日志, keydir中删除记录
	* 查
		+ 可能会从非active file中读取, 但是每次读都是读最新的版本, 所以系统中存在多个reader用于读取各自key的最新value
		+ 通过内存中的keydir找value所在的文件和offset
- 日志压缩
	* 新建一个active file, 赋予最新之上的版本号, 然后将keydir中关联的数据合并其中(原文是iterates over all non-active, 不过其实最终的状态就keydir中描述的), 最后可以删除过期的日志文件
	* 另外可以为该压缩创建一个"hint file"用于快速恢复keydir, 而不再需要扫描整个大的数据文件


- 优化
	* 索引可以用B Tree等结构


# 亮点

- hint file, 对某一时刻的"内存状态"做快照, 以快速恢复, 而不需要处理大文件
- 内存索引 + 外存数据













