---
title: acwing算法
author: 66RING
date: 2000-01-01
tags: 
- algorithm
mathjax: true
---

# Abstract


# Preface



## 贪心

- 区间问题
	* 套路: 
		1. 左端点排序或右端点排序
	* 证明套路: ans <= cnt + ans >= cnt => cnt = ans得整算法正确性
- 区间分组问题
	* 套路: 
		1. 左端点排序或右端点排序
		2. 可以使用最小堆等数据结构, 用一个"代表者"代替整个组, 就不用维护整个map
	* 证明套路: ans <= cnt + ans >= cnt => cnt = ans得整算法正确性


# misc

### 求质数

- 质除法
	* `i <= x/i`

```
bool is_prime(int x) {
	if (x < 2) return false; 
	for (int i = 2; i <= x/i; i++)
		if (x % i == 0)  {
			return false;
		}
	return true;
}
```

- 线性筛法



