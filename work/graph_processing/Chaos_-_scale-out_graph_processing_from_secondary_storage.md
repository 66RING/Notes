---
title: "Chaos: scale-out graph processing from secondary storage"
author: 66RING
date: 2022-08-01
tags: 
- graph processing
mathjax: true
---

# Abstract

- stream partition, 放弃内存locolity, 利用外存的优势
	* RAID like, 利用多台主机的吞吐量优势
	* 一个partition随机分散到各个主机 -> short pre-processing time
	* IO load balance by randomizing access ans location
- stealing work
	* 一个节点任务比较重时会自动让另一个replica负责



