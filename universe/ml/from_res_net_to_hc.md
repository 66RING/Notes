---
title: 从残差连接到Manifold-Constrained Hyper-Connections(mHc)
author: 66RING
date: 2026-01-03
tags: 
- machine learning
mathjax: true
---

# 从残差连接到mHc

## 标准残差

> 原始信息的直接传递

传统残差连接形如下，F表示Layer的fwd, wi表示fwd需要的权重。

$$
x_{i+1} = x_i + F(x_i, w_i)
$$

当多层layer堆叠时可以整理出公式：

$$
x_{1} = x_0 + F(x_0, w_0) \\
x_{2} = x_1 + F(x_1, w_1) \\
      = x_0 + F(x_0, w_0) + F(x_1, w_1) \\

x_{i} = x_0 + \sum_{0}^{i-1}{F(x_i, w_i)}
$$

可以看到**最浅层的x0的信息直通到了最深层xi**。

## Hyper-Connections

> 增加原始信息的带宽, 使其能传递更多信息

最浅层的信息就这么直接传最深层了，感觉信息量少了一点呢。那能不能让$xi = x0$先过一层高维度的隐状态再传入深层？

Hyper-Connections提出把信息传递的通道扩展n倍。

首先, 把残差中的x(1xd的向量)的通道扩展n倍成H(nxd的矩阵), 用H做残差。H = [x, x, x, x]

残差传播，引入几个可学习的矩阵: $H^{res}: (), H^{post}: (1,n), H^{pre}: (1,n)$。形式化表达如下：

$$
H_{i+1} = H^{res}_i H_i + H_i^{post^T} F(H_i^{pre} H_i, w_i)
$$


1. layer fwd: $F(H^{pre} H_i, w_i)$
    - $H^{pre}$聚合多通道信息再做fwd: 1xn @ nxd => 1xd
2. fwd后的结果扩展回多通道
    - $H^{post^T}$是一个nx1的矩阵, 所以有nx1 @ 1xd => nxd
3. 多通道残差连接
    - $H^{res}$是一个nxn的矩阵, 多维的聚合残差信息: nxn @ nxd = nxd

同理，我们可以整理出多层堆叠时的公式：

$$
H_1 = H^{res}_0 H_0 + H_0^{post^T} F(H_0^{pre} H_0, w_0) \\
H_1 = H^{res}_0 H_0 + \phi_0 \\
H_2 = H^{res}_1 H_1 + \phi_1 \\
H_2 = H^{res}_1 (H^{res}_0 H_0 + \phi_0) + \phi_1 \\
H_2 = (H^{res}_1 H^{res}_0) H_0 + (H^{res}_1 \phi_0 + \phi_1) \\
H_i = \prod_{k=0}^{i-1}H_k^{res} H_0 + \sum_{k=0}^{k=i-1}{(\prod_{j=k+1}^{i-1}H_j^{res})\phi_k} \\
$$

此时, 从最浅层H0到到最深层Hi的信息传递通过一种多维矩阵(nxn)的连乘进行。

$$
H_i = \prod_{k=0}^{i-1}H_k^{res} H_0 + ...
$$


## Manifold-Constrained Hyper-Connections(mHc)

> Deepseek提出的mHc
>
> 限制信息的缩放

hc通过矩阵连乘的方式传递信息，**这种连乘是没有约束的**，会导致信息的无界放大/衰减。

$$
H_i = \prod_{k=0}^{i-1}H_k^{res} H_0 + ...
$$

所以mhc提出给$H^{res}$加点约束。

1. Birkhoff多胞形: **每行的和是1, 没列的和是1, 所有元素非负**
    - 好处: (1) norm preservation, 谱范数不大于1, 不会信号放大 (2) composition closure, 乘法是封闭的, 多个多胞形连乘的依旧是一个多胞形空间 (3) 是一种能量守恒的特征融合(加权和)
2. Sinkhorn-Knopp算法: 多胞形求解算法 (迭代算法)
    1. 通过指数运算把所有元素编程正数
    2. 迭代: 交替缩放行和列 (迭代20次)
        - 缩放方法: 所有元素除以所有元素之和

