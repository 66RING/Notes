---
title: Log Structure Storage
author: 66RING
date: 2022-08-01
tags: 
- data structure
mathjax: true
---

# Abstract

- log
	* 老版本保留, 引用改变
	* root node总是插入在头, 维护引用
- RCU
	* 指针修改
	* 因为老版本保留, 不会存在数据竞争, 最多访问"过时"
- 循环buffer, 回收老版本
	* 再不济就垃圾回收
	* 分chunck回收

http://blog.notdot.net/2009/12/Damn-Cool-Algorithms-Log-structured-storage

# Preface


# Overview
