---
title: 闫氏DP分析法
author: 66RING
date: 2000-01-01
tags: 
- algorithm
mathjax: true
---

# Abstract

从集合的角度思考问题

# 动态规划

- 动态规划
	* 状态表示`dp(i, j, k, l, ...)`
		+ 集合: 想办法归类, e.g. 走到`[i, j]`的路线, `[i, j]`i天j人的情况
		+ 属性: 常见的就MAX/MIN/数量
	* 状态计算: 集合划分, 划分后的集合分别怎么计算得到
		+ 一个重要的依据: "最后"
		+ **不吝啬特判**
		+ e.g. 来自哪里


