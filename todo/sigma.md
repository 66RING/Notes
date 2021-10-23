---
title: Sigma求和
date: 2021-10-22
tags: 
- tips
mathjax: true
---

# Abstract

> 改写求和，实际上是在改写程序

数学作为抽象工具，替代不可靠的直觉

# Preface


# Overview

# eg

## eg1 求等差数列

1 + 2 + 3 + ... + n可以写成：

$$\sum_{1 \le k \le n} k \tag{1.1}$$

相当于程序中的循环，而利用高斯的技巧我们利用**对称性**可以换元成：n + n-1 + ... + 1:

$$\sum_{1 \le k \le n} n+1-k \tag{1.2}$$

公式1.1和1.2是同一个集合求和，于是我们利用结合律可以式1.1加式1.2得：

$$\sum_{1 \le k \le n} (k) + (n+1-k) \tag{1.3}$$

$$\sum_{1 \le k \le n} n+1 \tag{1.4}$$

1.4中变量n+1和集合k没有关系，相当于常量，可以分配律提出:

$$(n+1)\sum_{1 \le k \le n} = (n+1) n \tag{1.5}$$

所以得等差数列

```
Sigma -> 求和 -> 循环 -> 变换 -> 优化循环!
```

## eg2 $\sum_{1 \le k \le n} k \cdot 2^k$

1. 换元$p(k) = k+1$
2. TODO整理一下
3. 得Sn和Sn-1的关系!!


# 多重求和

> 多重循环

## 上三角形求和

```
x x x
. x x
. . x
```

$$\sum_{1 \le i \le n} \sum_{1 \le j \le n} a_i b_j \tag{2.1}$$

上述双重循环可以优化成两个一重循环(常量提出)

$$(\sum_{1 \le i \le n} a_i) (\sum_{1 \le j \le n} b_j) \tag{2.2}$$

那么$\sum_{1 \le i \le j \le n} a_i b_j$呢？这个式子相当于上三角形的求和

引入一个新的工具中括号条件运算符：`[]`，true时返回1, 否则0。因此我们可以改写上式这个不规则的求和为:

$$\sum_{1 \le i \le n} \sum_{1 \le j \le n} a_i b_j[i \le j] \tag{2.3}$$

再利用**对称性**的技巧想办法把它弄得"整齐"：上三角形求和 + 下三角形求和

$$S_上 + S_下 = \sum_{1 \le i \le n} \sum_{1 \le j \le n} a_i b_j([i \le j] + [j \le i])$$

那么就优化成了一个矩形求和+对角线求和：

$$\sum_{1 \le i \le n} \sum_{1 \le j \le n} a_i b_j + \sum_{1 \le i \le n} \sum_{1 \le j \le n} a_i b_j[i = j]$$
$$(\sum_{1 \le i \le n} a_i) (\sum_{1 \le j \le n} b_j) + \sum a_i^2\tag{2.2}$$

二重求和巧妙地变成了一重求和，**严格的数学分析，而不是脑子里不太可靠的直觉!!**


**$\sum_{1 \le k \lt n} \lfloor \sqrt{k} \rfloor$**
========

怎么优化这个程序呢?

tips: 求和中的元素比较少的话，可以通过枚举的方式进行优化

