---
title: welford算法求方差
author: 66RING
date: 2025-12-03
tags: 
- gpu
- hpc
mathjax: true
---

# welford算法求方差

> cc https://zhuanlan.zhihu.com/p/408474710

## 基础方法

- $D(x) = \frac{\sum(x_i - mean)^2}{n}$
    * 需要两次遍历1. 第一次遍历获取mean, 2. 第二次遍历计算方差

## 数学变换优化

- 数学等价转换的一次遍历
    * 可以推导出$D(x) = E(x^2) - E(x)^2$, 只需要用x和x方做一次遍历即可
    * 但是先累加容易出现精度问题

$$
D(x) = \frac{\sum(x_i - u)^2}{n} \\
     = \frac{\sum(x_i^2 - 2ux + u^2)}{n} \\
     = \frac{\sum(x_i^2) - \sum(2ux) + nu^2}{n} \\
     = E(x^2) - 2uE(x) + u^2 \\
     = E(x^2) - E(x) \\
$$


## welford

> 数学变化, 转换成迭代的表达式, e.g. x_{n+1} = ...x_{n}

结论:

$$
E_{n+1}(x) = E_{n}(x) + \frac{x_{n+1} - E_n(x)}{n+1}
$$

$$
D_{n+1}(x) = D_{n}(x) + \frac{(x_{n+1} - E_{n}(x))(x_{n+1}-E_{n+1}(x)) - D_{n}(x)}{n + 1}
$$

其中$E_n(x)$和$D_n(x)$分别表示对前n个x求的均值和方差

## welford误差优化

额外记录一个M2值, M2更新公式:

$$
M2_{n} = M2_{n-1} + (x_n - E_{n-1}(x))(x_n - E_n(x))
$$

最后用M2来计算方差

$$
D(x) = \frac{M2_n}{N}
$$


## impl

```python
import numpy as np

def welford_update(count, mean, M2, currValue):
    count += 1
    delta = currValue - mean
    mean += delta / count
    delta2 = currValue - mean
    M2 += delta * delta2
    return (count, mean, M2)


def naive_update(sum, sum_square, currValue):
    sum = sum + currValue
    sum_square = sum_square + currValue * currValue
    return (sum, sum_square)


x_arr = np.random.randn(100000).astype(np.float32)

welford_mean = 0
welford_m2 = 0
welford_count = 0
for i in range(len(x_arr)):
    new_val = x_arr[i]
    welford_count, welford_mean, welford_m2 = welford_update(welford_count, welford_mean, welford_m2, new_val)
print("Welford mean: ", welford_mean)
print("Welford var: ", welford_m2 / welford_count)

naive_sum = 0
naive_sum_square = 0
for i in range(len(x_arr)):
    new_val = x_arr[i]
    naive_sum, naive_sum_square = naive_update(naive_sum, naive_sum_square, new_val)
naive_mean = naive_sum / len(x_arr)
naive_var = naive_sum_square/ len(x_arr) - naive_mean*naive_mean
print("Naive mean: ", naive_mean)
print("Naive var: ", naive_var)
```
