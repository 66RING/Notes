---
title: leveldb读写操作笔记
author: 66RING
date: 2023-10-01
tags: 
- database
- leveldb
mathjax: true
---

# leveldb读写操作笔记

> https://leveldb-handbook.readthedocs.io/zh/latest/rwopt.html

## 整体架构

- 存内数据结构
    * MemTable
        + 一种有序的存内结构(跳表)
        + 写入先写入memtable, 当内容达到阈值后将其转换成immutable memtable
    * Immutable MemTable
        + 只读的memtable, 可以用来**后台压缩**
- 文件系统
    * log
    * manifest
        + manifest文件就是用来记录这些versionEdit信息的





