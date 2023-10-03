---
title: LSM tree笔记
author: 66RING
date: 2023-09-29
tags: 
- db
mathjax: true
---

# LSM tree笔记

- 数据结构:
    * 一个内存中的有序结构: level 0
        + 排序树(MemTable)
    * 一个外存中的append only结构: level 1+n
        + 每个level内可以有多个SSTable, SSTable内有序
            + 一个level多个SSTable是一种Tiered compaction的设计, 有多次merge的设计
        + 内存中排序好的数据顺序写入磁盘
    * 每个level达到阈值后合并(多路归并排序)到下个level中

重点在与SSTable的设计

## 增删改查

> 修改都只用操作内存表, log struct会覆盖历史记录

- 插入
    * 插入内存结构中
- 删除: 都是在level0中插入删除标记。使用墓碑标记是因为每个level中都可能存在数据的记录
    * 目标数据在内存中, 直接在内存中标记删除
    * 目标数据在磁盘中, 在内存中**插入**删除标记
    * 目标数据不存在, 在内存中**插入**删除标记
- 修改: 直接操作内存表, 覆盖或者插入即可
    * 目标数据在内存中, 直接在内存中修改, 覆盖即可
    * 目标数据在磁盘中, 在内存中**插入**新数据
    * 目标数据不存在, 在内存中**插入**新数据
- 查询: 按level顺序逐级查找, 找到就返回
    * level0: 排序树的查找
    * lelve1+n: 逐级查找, 可以利用有序结构做二分等

## 合并

有两种类型的合并

- 内存数据保存到磁盘
    * 内存数据达到阈值后顺序写入level0的一个新SSTable
- 磁盘数据合并到下一个level
    * 一个level达到阈值后使用多路归并排序将SSTable合并到下一个level的新SSTable中
        + 游标先游走较新的SSTable, 保证相同的元素会被较新的SSTable覆盖


## 优化

各种压缩(合并)策略

- Classic leveled compaction
    * N个level, 每个level一个SSTable, level间的SSTable往往成倍数关系(fan out, 扇出)
    * 当一个level到达数据量阈值时触发压缩压缩到下一个level
    * Li层数据与Li+1层数据在row key有交集的地方进行压缩, 得到新的Li+1层数据
    * 雪崩效应: 如果每个level都快到达了阈值, 一次压缩就导致多层都触发压缩, 加剧**写放大**
- Size-Tiered compaction
    * N个level, 每个level多个SSTable
    * 统一level中的SSTable可能存在交集, 扫描时需要多个SSTable, 导致**读放大** (读到重复数据)
    * 压缩: 同级的所有SSTable压缩, 但不与i+1层SSTable压缩, **减缓写放大**
- Tiered & Leveled compaction(混合模式)⭐
    * 层级低的level使用leveled模式, 层级高的leveled使用tiered模式
- FIFO compaction
    * 不分level, SSTable维护在一个FIFO链表中, 链尾会被淘汰
    * 简单但会丢数据, 适合时序数据库等时间敏感



