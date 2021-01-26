---
title: 用processing模拟自然系统
date: 2020-4-14
tags: 
- processing
- nature of code
---

## 随机游走

- processing中的random()函数生成的随机数是均匀分布的
- 可以利用random函数生成非均匀分布的随机数
    - 创建一个非均匀的数组，用random来随机的取索引
- 也可用random来取概率，但是注意数据类型(float和int有区别)
- 随机数的正态分布
    - `(float)generator.nextGaussian()`返回一个高斯随机数，nextGaussian返回值的类型是double


### Perlin噪声(一种更平滑的算法)

Perlin生成的随机数更平滑，但仍有一定的随机性。

Processing内置了`noise()`函数，可以有1~3个参数。
分别表示一维、二维、三维随机数。

noise函数返回的结果总是在0~1之间，我们可以通过map函数来改变结果的范围，
可以吧一维的Perlin噪声当作随着时间推移而发生变化的线性序列。
通过往noise传入一个"指定的时间"来获取这个时间点上的噪声。

同理，二维噪声像个崎岖不平的面


### map映射

`map(a, b, c, d)`, 原范围(a, b), 希望映射到的范围(c, d)


## 向量




