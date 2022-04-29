---
title: Raft牺牲了什么
author: 66RING
date: 2022-04-21
tags: 
- distributed
mathjax: true
---

# Abstract

思维方式，系统设计应该考虑到的问题

优化的基本思路：

- 一次一batch
- 有共享就有瓶颈分析
- log order如何利用多核性能

# Overview

- batch write back
- single AppendEntries
- snaptshot only in small state
- recovery by snapshot, slow and some out of date
- low multi-core advantage: operation executed in log order

都需要魔改


# References

- [Does Raft sacrifice anything for simplicity?](https://pdos.csail.mit.edu/6.824/papers/raft-faq.txt)
