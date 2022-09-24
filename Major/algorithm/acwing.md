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


# Algo

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


## 搜索与图论

- 最短路径问题
	* 单源
		+ 不存在负权边
			+ 朴素dijkstra O(n**2)
			+ 堆优化dijkstra O(m*logn)
		+ 存在负权边
			+ bellman-ford O(mn)
			+ spfa 一般O(m), 最坏O(mn)
	* 多源
		+ floyd算法

### 图的邻接表表示

```c
// 添加a->b的链接表, 权值为c
// 这里用数组模拟, idx就相当于内存位置
// h[]是整个"结构体"的如何, 存放编号到内存地址(idx)的映射
void add(int a, int b, int c) {
  e[idx] = b; // 添加新节点
  w[idx] = c; // 添加节点权值
  ne[idx] = h[a]; // 头插链边
  h[a] = idx++;  // 更新头节点
}
```


### bellman-ford

可以处理有负权边的最短路问题

- algo: 单源负权边最短路问题, 多次遍历所有边即可。
	* TODO: 待证明
- 更新只用上一次的结果, 防止使用修改的后的进行计算
- 三角不等式
- 之所以可以解决负边问题是因为循环次数有限


### spfa算法

- 同样是解决存在负权边的最短路径问题。
- 是对bellman-ford的优化, bellman不假思索遍历所有, 但实际上如果dist没变那依赖它跳转的后续就不会改变
	* 所以只需要在dist缩小时才将它加入队列

举个例子, bellman-ford算法伪代码如下:

```c
for 所有点k(每个点都可能称为中介跳转):
	for 所有点i(枚举所有起点, 使用中介点k relax):
		relax(dist)
```

spfa伪代码如下

```c
exist[1] = true; 	// 是否存在队列中
for 所有有在队列q中的点k: 	// 使用一个队列保存dist缩小后的点
	exist[k] = false; 	// 清除记录, 后续必要时会重新添加
	for 所有k的邻接点i(枚举所有起点, 使用中介点k relax):
		if 更新:
			if !exist[j] q <- j; // 如果已经在组列中就不如
```


#### spfa判断负环

原理: 

1. n个点不存在环的话只有n-1条边, 所以如果`cnt[x] >= n`则说明存在环路。`cnt[x]`表示走到x所需的边数。`cnt[x] = cnt[prev(x)] + 1`, 一次走一步加一边。
2. 因为负环会让路径变小, 所以会被检测到


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


