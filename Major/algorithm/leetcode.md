---
title: leetcode算法笔记
author: 66RING
date: 2000-01-01
tags: 
- algorithm
mathjax: true
---

# Abstract


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

[link](https://leetcode-cn.com/problems/minimum-number-of-arrows-to-burst-balloons/description/)

算法同[无重叠区间](#无重叠区间)。思考**有多少交集**，贪心：从首区间开始**交集越多越好**，没有交集的下一个区间称为新的首区间。



### 最长递增子序列

[300.最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

TODO


### 406. 根据身高重建队列

[link](https://leetcode-cn.com/problems/queue-reconstruction-by-height/)

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


## 二分查找

### 69. x 的平方根 

[link](https://leetcode-cn.com/problems/sqrtx/description/)

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

[link](https://leetcode-cn.com/problems/find-smallest-letter-greater-than-target/description/)

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

[link](https://leetcode-cn.com/problems/single-element-in-a-sorted-array/description/)

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

[link](https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array/description/)


### 34. 在排序数组中查找元素的第一个和最后一个位置

[link](https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

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

[link](https://leetcode-cn.com/problems/different-ways-to-add-parentheses/description/)

TODO redo **COOL**

### 95. 不同的二叉搜索树 II

[link](https://leetcode-cn.com/problems/unique-binary-search-trees-ii/description/)

TODO redo **COOL**


## 动态规划

### 无重叠区间

[leetcode](https://leetcode-cn.com/problems/non-overlapping-intervals/)

TODO

### 最长递增子序列

[300.最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

`dp[i]`表示`[:i]`的最长递增子序列。如果`nums[j] > nums[i]`，i为`[0,j)`说明`dp[j]`至少可以是`dp[i]+1`。

- `for i in 0..j; dp[j] = max(dp[j], dp[i]+1)`
	* 当j前的nums[i]比nums[j]小时，dp[j]可能增加

### 70. 爬楼梯

[link](https://leetcode-cn.com/problems/climbing-stairs/description/)

TODO redo

### 198. 打家劫舍

TODO redo

[link](https://leetcode-cn.com/problems/house-robber/description/)

### 213. 打家劫舍 II

TODO redo

[link](https://leetcode-cn.com/problems/house-robber-ii/description/)


### 64. 最小路径和

求左上到右下的最短路径

[link](https://leetcode-cn.com/problems/minimum-path-sum/description/)

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

[link](https://leetcode-cn.com/problems/unique-paths/description/)

- 类似64. 最小路径和
- tips: 先把边缘初始化了，剩下的遍历方便点


### 303. 区域和检索 - 数组不可变

[link](https://leetcode-cn.com/problems/range-sum-query-immutable/description/)













