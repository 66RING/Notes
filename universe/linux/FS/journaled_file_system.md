---
title: 日志型文件系统
date: 2021-03-03
tags: 
- linux
- kernel
- filesystem
mathjax: true
---

# 日志型文件系统

试想这么一个场景，磁盘正在写data block或data bitmaps或inodes时发生了crash，要如何保证数据的一致性呢？这时使用 **日志journaling** 就是一个很好的解决方案。


## 原理

在真正更新磁盘数据前，会先往磁盘写入一些信息，这些信息用于描述接下来的任务，这种方式被称为 **write-ahead loggind**。

当发生crash时就可以通过记录的信息回溯crash前的操作，称为 **replay**。

为了支持日志，日志会同data block一样占用一部分磁盘空间。

我们会把一个完整的更新操作的步骤(更新inode, bitmap, data block)称为一个 **事务(transcation)** 。会在transcation开始和结束加上TxB和TxE用于判断transcation是否完整。在transcation完成之后才真正更新磁盘数据，这步称为 **checkpoint**。

由于调度等原因，各个部分不一定会按顺序写入，即TxB和TxE都写了但其之间的信息(inode, bitmap, data block)没写。为了保证TxE是在步骤信息写入完成后才写的，需要使用 **write barrier** 机制。写入除TxE之外的部分叫做 **journal write**。journal write完成后再写入TxE( **journal commit** )。

如果不能接受barrier对性能的影响，可以在挂载文件系统的时候使用"nobarrier"选项。

写操作完成后就可以释放journal占用的空间了。

## 优化

一直方法是将多个journal合并处理

**Writeback模式**，journal只记录meta data(inode和bitmap)的操作步骤

这样不用记录user data的模式减小的journal的开销，但是user data可能会在任何时间写入，如在journal写操作之后，写user data之前，这样就会造成数据的不一致。但这比不用journal的危害要低，算是一种性能和稳定性的平衡吧。

**Ordered模式**，在写user data之后才写journal。这样可能会造成数据的丢失，但是不会造成数据不一致。



