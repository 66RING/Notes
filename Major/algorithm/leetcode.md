---
title: leetcode算法笔记
author: 66RING
date: 2000-01-01
tags: 
- algorithm
mathjax: true
---

# Abstract

TODO learn 贪心 剪枝 dp 的思想和trade off

# Preface


# Overview

## 贪心法

### 无重叠区间

[leetcode](https://leetcode-cn.com/problems/non-overlapping-intervals/)

思考会议预约，区间`(start, end)`分别表示会议开始时间和会议结束时间。**贪心：会议越早结束，留给后面分配的时间越多**。

- 所以**根据结束时间(右值)排序**

如果情况如下, 那我不能删除R0，因为你R1占用更多时间。

```
(R0  )
	(R1    )

等价

	(R1    )
(R0  )
```

如果情况如下，那不重叠区间可以++，更新结束时间为R1的结束。

```
(R0  )
		(R1    )
```

### 452. 用最少数量的箭引爆气球

TODO redo

[leetcode](https://leetcode-cn.com/problems/minimum-number-of-arrows-to-burst-balloons/description/)

算法同[无重叠区间](#无重叠区间)。思考**有多少交集**，贪心：从首区间开始**交集越多越好**，没有交集的下一个区间称为新的首区间。



### 最长递增子序列

[300.最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

TODO


### 406. 根据身高重建队列

[leetcode](https://leetcode-cn.com/problems/queue-reconstruction-by-height/)

- 当按照身高排序时，然后插入。因为按身高从高到低排序，那么后插入的身高一定比前面出现的身高都低，那么它插入时就可以根据"它前有多少人"自由选择位置，因为都比它高。后面插入的也不会它，因为后面插入的一定比它矮
- 如果存在相同身高，那么"前面比他高"的人数"小"的先选择插入，因为"前面比他高"的值大的一定插入在后面，不会应该前面插入的值


### 121. 买卖股票的最佳时机

TODO 怎么说呢

```cpp
int maxProfit(vector<int>& prices) {
	int minprice = INT_MAX;
	int size = prices.size();
	int res = 0;
	for(int i=0; i<size; i++) {
		res = max(res, prices[i] - minprice);
		minprice = min(prices[i], minprice);
	}

	return res;
}
```

### 122. 买卖股票的最佳时机 II

贪心：**能涨一点算一点**

```
int maxProfit(vector<int>& prices) {   
	int ans = 0;
	int n = prices.size();
	for (int i = 1; i < n; ++i) {
		ans += max(0, prices[i] - prices[i - 1]);
	}
	return ans;
}
```


### 53. 最大子数组和

如果`nums[i]`比`nums[i]+prefix`大, 那prefix从`nums[i]`重新开始


### 763. 划分字母区间

遍历字符串, 找每个字符能出现的最右端。一直找最右不断更新, 当`i == end`时区域结束

```c
for(int i=0; i<size; i++) {
	map[s[i]-'a'] = i;
}
```


## 二分查找

### 69. x 的平方根 

[leetcode](https://leetcode-cn.com/problems/sqrtx/description/)

```cpp
int mySqrt(int x) {
	int l = 0, r = x, ans = -1;
	while (l <= r) {
		int mid = l + (r - l) / 2;
		if ((long long)mid * mid <= x) {
			ans = mid;
			l = mid + 1;
		} else {
			r = mid - 1;
		}
	}
	return ans;
}
```

### 744. 寻找比目标字母大的最小字母

[leetcode](https://leetcode-cn.com/problems/find-smallest-letter-greater-than-target/description/)

TODO ??

```cpp
char nextGreatestLetter(vector<char>& letters, char target) {
	int l=0, h = letters.size();
	int mid;
	int size = letters.size();
	while(l < h) {
		mid = (l+h)/2;
		if(letters[mid] > target) {
			h = mid;
		} else if (letters[mid] <= target) {
			l = mid+1;
		}
	}
	return letters[l % size];
}
```


### 540. 有序数组中的单一元素

[leetcode](https://leetcode-cn.com/problems/single-element-in-a-sorted-array/description/)

- **数组长度必是奇数**
- 如果偶数下标等于偶数下标+1处的值，光棍在右边。否则在左边，移动偶数下标位
- 如何结束，如何状态转移??

```cpp
int singleNonDuplicate(vector<int>& nums) {
	int l=0, h = nums.size()-1;
	int mid;
	while(l < h) {
		mid = (l+h)/2;
		mid -= mid&1;
		if(nums[mid] != nums[mid+1]) {
			h = mid;
		} else {
			l = mid+2;
		}
	}
	return nums[l];
}
```

### 153. 寻找旋转排序数组中的最小值

[leetcode](https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array/description/)


### 34. 在排序数组中查找元素的第一个和最后一个位置

[leetcode](https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

**COOL**

- 根据结束位置和开始位置判断转移, e.g.
	* `while(l < r)`结束时必有`l == r`
	* `nums[mid] < target`中间位置和中间位置左边一定不是开始位置，`l = mid + 1`
	* `nums[mid] == target`中间位置左边一定不是结束位置，`l = mid`
	* 分析三个分支不丢人(`> < ==`)，更容易分析，然后再合并优化
	* 因为存在`l=mid`，求结束位置的**求mid要上取整**: `mid = (r+l+1)/2`
	* 可以额外判断越界


## 分治

### 241. 为运算表达式设计优先级

[leetcode](https://leetcode-cn.com/problems/different-ways-to-add-parentheses/description/)

TODO redo **COOL**

### 95. 不同的二叉搜索树 II

[leetcode](https://leetcode-cn.com/problems/unique-binary-search-trees-ii/description/)

TODO redo **COOL**


## 动态规划

动态规划的本质就是枚举找最优

- 确定base case
- 确定状态(转移)
	* 数学归纳
- 确定选择
- 定义dp

1. 暴力地
2. 带"备忘录"的


### 二维压缩到一维防覆盖的办法


### 无重叠区间

[leetcode](https://leetcode-cn.com/problems/non-overlapping-intervals/)

TODO

### 最长递增子序列

[300.最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

`dp[i]`表示`[:i]`的最长递增子序列。如果`nums[j] > nums[i]`，i为`[0,j)`说明`dp[j]`至少可以是`dp[i]+1`。

- `for i in 0..j; dp[j] = max(dp[j], dp[i]+1)`
	* 当j前的nums[i]比nums[j]小时，dp[j]可能增加

### 70. 爬楼梯

[leetcode](https://leetcode-cn.com/problems/climbing-stairs/description/)

TODO redo

### 198. 打家劫舍

TODO redo

[leetcode](https://leetcode-cn.com/problems/house-robber/description/)

### 213. 打家劫舍 II

TODO redo

[leetcode](https://leetcode-cn.com/problems/house-robber-ii/description/)


### 64. 最小路径和

求左上到右下的最短路径

[leetcode](https://leetcode-cn.com/problems/minimum-path-sum/description/)

从终点到起点dp。e.g.

```
1 2 3
4 5 6
按6->1是顺序dp
```

- 因为只可以向右或向下走，思路就是保证dp的时候下方和右方都已知最小。
- 最后dp可以优化成一维数组
- tips: 先把边缘初始化了，剩下的遍历方便点


### 62. 不同路径

左上到右下的不同路径数，只能向右和向下走

[leetcode](https://leetcode-cn.com/problems/unique-paths/description/)

- 类似64. 最小路径和
- tips: 先把边缘初始化了，剩下的遍历方便点


### 303. 区域和检索 - 数组不可变

[leetcode](https://leetcode-cn.com/problems/range-sum-query-immutable/description/)

- 前缀和相减
- tips因为是左闭右闭的，可以后移1个，申请`n+1`个单元


### 413. 等差数列划分

[leetcode](https://leetcode-cn.com/problems/arithmetic-slices/description/)

TODO redo

### 背包问题

[acwing](https://www.acwing.com/problem/content/2/)



### 322. 零钱兑换

[leetcode](https://leetcode-cn.com/problems/coin-change/description/)

**COOL** TODO redo

- dp

`dp[i]`表示能够凑够i所需的最少硬币数，枚举硬币类型`coins[j]`，如果用了`coins[j]`面值的硬币，那么这种情况的硬币数应该为`dp[i-coins[j]] + 1`


### 518. 零钱兑换 II

[leetcode](https://leetcode-cn.com/problems/coin-change-2/description/)

求能凑出amount的组合数


### 309. 最佳买卖股票时机含冷冻期

[leetcode](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-cooldown/description/)


### more dp 

https://github.com/CyC2018/CS-Notes/blob/master/notes/Leetcode%20%E9%A2%98%E8%A7%A3%20-%20%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92.md#1-%E7%88%AC%E6%A5%BC%E6%A2%AF

### dp思想

1. 子问题最优解



## 搜索

- dfs, 递归
- bfs, queue

### 1091. 二进制矩阵中的最短路径

[leetcode](https://leetcode-cn.com/problems/shortest-path-in-binary-matrix/)

DFS vs BFS

- 单可能 -> DFS, 因为多可能时大量回溯
- 多可能 -> BFS

相当于用BFS从起点不断外扩的过程，都把queue走完(边界外扩)。外层循环的次数就是最短路径。

```
e.g

0 1 x x
1 1 x x
x x x x
x x x x

0 1 2 x
1 1 2 x
2 2 2 x
x x x x

```

**注意是可以斜着走的**


### 279. 完全平方数

[leetcode](https://leetcode-cn.com/problems/perfect-squares/description/)

> 给你一个整数 n ，返回 和为 n 的完全平方数的最少数量 。

TODO redo

- dp
	* `dp[i]`表示组成i使用的完全数的最少数量
	* 局部最优到整体最优：从小到大算，`dp[i-j*j]`最少已知了，那用上`j*j`这个数的话数量就是`dp[i-j*j]+1`


### 695. 岛屿的最大面积

[leetcode](https://leetcode-cn.com/problems/max-area-of-island/description/)

- bfs/dfs

### 200. 岛屿数量

[leetcode](https://leetcode-cn.com/problems/number-of-islands/)

- 同上dfs/bfs
	* dfs, 递归
	* bfs, queue
- 可以复用grid做visited


### 130. 被围绕的区域

[leetcode](https://leetcode-cn.com/problems/surrounded-regions/description/)

**COOL**：不被包围则必定直接或间接地与边界相连。从边界`O`点dfs/bfs，标记与直接间接相连的点。再整个遍历填充。

遍历四周的tips

```
[i, 0]
[i, m-1]
|	|
V	V

[0,   i]
[m-1, i]
->

->
```

### 回溯

- dfs
- 常用于解决**排列组合**问题
- dfs不立即返回，注意元素的标记问题


### 17. 电话号码的字母组合

[leetcode](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/description/)

dfs，尝试一个字母后回溯，再试下一个

```
for(char x: phoneMap.at(digits[index])) {
	combination.push_back(x);
	backtrack(combinations, phoneMap, digits, index+1, combination);
	combination.pop_back();
}
```


### 93. 复原 IP 地址

[leetcode](https://leetcode-cn.com/problems/restore-ip-addresses/description/)

- `str.substr(start, len)`截取字串
- string可以用`+=`，`substr()`和`pop_back()`增删
- 模板`(combination, &combinations)`, combination表示当前状态，combinations用于接收完成的combinations，递归，然后回溯
	* combination用引用和不用引用根据实际情况定



### 257. 二叉树的所有路径

[leetcode](https://leetcode-cn.com/problems/binary-tree-paths/description/)

- combination不用引用，这样函数返回时就不用考虑回溯问题


### 37. 解数独

TODO hard

[leetcode](https://leetcode-cn.com/problems/sudoku-solver/description/)


### 15. N 皇后

TODO hard

[leetcode](https://leetcode-cn.com/problems/n-queens/description/)












