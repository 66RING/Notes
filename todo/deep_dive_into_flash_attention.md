---
title: 深入FlashAttention(源码解读和魔改)
author: 66RING
date: 2023-12-06
tags: 
- machine learning
- machine learning system
mathjax: true
---

# Abstract

https://blog.51cto.com/u_15302822/7916287

1. 为什么要safe softmax
    - 为什么safe softmax和native等价: 指数部分加减相当于底数乘除
        * $y_i = \frac{e^{x_i - max(x)}}{\sum_j{e^{x_j - max(x)}}} = \frac{e^{-max(x)} \times e^{x_i}}{e^{-max(x)} \times \sum_j{e^x_j}}$
2. 简易safe softmax实现
3. 为什么要tilling
    - 每次都有读入整个数据(max, sum等)
4. 简易online safe softmax实现
5. 简易flashattention实现


## impl

- NOTE
    * pytorch中的乘法
        + TODO


@ 和 *代表矩阵的两种相乘方式：@表示常规的数学上定义的矩阵相乘；*表示两个矩阵对应位置处的两个元素相乘。

x.dot(y): 向量乘积,x，y均为一维向量。

*和torch.mul()等同:表示相同shape矩阵点乘，即对应位置相乘，得到矩阵有相同的shape。

@和torch.mm(a, b)等同：正常矩阵相乘，要求a的列数与b的行数相同。

torch.mv(X, w0):是矩阵和向量相乘.第一个参数是矩阵，第二个参数只能是一维向量,等价于X乘以w0的转置
