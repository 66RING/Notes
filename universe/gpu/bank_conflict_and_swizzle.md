---
title: bank confict和冲突消解
author: 66RING
date: 2024-08-10
tags: 
- gpu
- hpc
mathjax: true
---

# bank confict和冲突消解

> 一个conflict的实例: 矩阵转置存储在smem中。 thr0~3读取gmem一行, 存储到smem的一列, 这时同一列的thr就发生列bank conflict

- bank conflict
    * 4Byte一个bank
- 简单方法
- ldmatrix
- swizzle

GPU为了提升并行度，可以提供了同时访问share memory功能，多个线程访问smem的不同bank可以并行，N个线程访问同一个bank就会串行执行，这就是bank conflict，称为N路bank conflict。

假设GPU中4 Byte一个bank，以一个实际的矩阵乘法的场景为例分析bank conflict和解决方法。假设一个1x32 @ 32x1的矩阵乘法：

```
__share__ float A[32][32];
__share__ float B[32][32];

A: 
0 1 ... 31

B:
0 -> T0
1 -> T1
..
31 -> T31
```

矩阵A可以32个线程无冲突的读取数据，而矩阵B的每个线程都会访问同一个bank，会产生bank conflict。

解决bank conflict的本质就是重新映射存储的物理地址，使得每个线程访问的数据不在同一个bank上。

## 简单方法

一个简单的解决方法是增加smem的容量，如把`B[32][32]`改成`B[32][33]`, 这样一来T0访问bank 0，因为最后空出了一个bank，T1就访问到了bank 1，以此类推，从而解决了bank conflict。


## 向量指令的复杂场景

当来到更复杂的场景，这么简单的扩容就不太好用了。比如使用向量指令(ldmatrix)一次读取多个数据时，那么只要这个vector有一个bank冲突就导致整个读取冲突。

当然使用增加smem大小的方法也是可行的，比如把向量指令的长度看成一组去做扩容，这样两组向量指令之间就会offset 1来解决bank conflict。

cutlass中引用了swizzle的方法来解决更复杂的场景。


## **swizzle**

swizzle使用BMS描述，表示有$2^B$行$2^S$列, $2^M$个元素为一组。

使用bit-wise的异或算法来规划每个数据的线程号。如下就是一个SMS=(3, 0, 3)的swizzle。

| row\col | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---------|---|---|---|---|---|---|---|---|
| 0       | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| 1       | 1 | 0 | 3 | 2 | 5 | 4 | 7 | 6 |
| 2       | 2 | 3 | 0 | 1 | 6 | 7 | 4 | 5 |
| 3       | 3 | 2 | 1 | 0 | 7 | 6 | 5 | 4 |
| 4       | 4 | 5 | 6 | 7 | 0 | 1 | 2 | 3 |
| 5       | 5 | 4 | 7 | 6 | 1 | 0 | 3 | 2 |
| 6       | 6 | 7 | 4 | 5 | 2 | 3 | 0 | 1 |
| 7       | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |

"swizzle: $2^M$个元素一组做扭转"

+ "两两拧麻花"
+ **y_p=y_l: 物理行等于逻辑行, x_p = x_l ^ y_l: 物理列等于(逻辑列 xor 逻辑行)**
+ 原理: **物理地址的重新映射**
    + src和dst怎么对应?
        + 访问使用的是逻辑地址: i行j列, 当访问时的物理地址会用i,j重新映射
+ 原理: 任意行不存在相同的两个元素(物理列), 任意列不存在相同的两个元素(物理列)
    + 任意行: x_p1 = x_l1 ^ y, x_p2 = x_l2 ^ y, x_p1 ^ x_p2 = (y^y)^(x_l1^x_l2) = 0^(x_l1^x_l2) != 0
        + 因为x_l1 != x_l2, 所以x_l1^x_l2 != 0
    + 任意列: 同理x_p1 = x^y_l1, x_p2 = x^y_l2, x_p1 ^ x_p2 = (x^x)^(y_l1^y_l2) = 0^(y_l1^y_l2) != 0
        + 因为y_l1 != y_l2, 所以y_l1^y_l2 != 0





