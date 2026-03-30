---
title: SVD奇异值分解
author: 66RING
date: 2024-10-30
tags: 
- machine learning
mathjax: true
---

# SVD奇异值分解

> `torch.svd()`即可

- https://www.bilibili.com/video/BV1ExWxesEVf/?vd_source=fa5227c06f0a0c9f870b406f10be1d31
- https://medium.com/@weidagang/low-rank-approximation-with-svd-in-numpy-4aee9077eadf

原始矩阵(M)可以分解成三个矩阵的相乘: $M = U \Sigma V^T$, M = 旋转, 拉伸, 旋转

- 对角矩阵, 拉伸, 只在对角线上有值
    * 左乘时起到一个拉伸的作用
    * 维度消除/增加
- 正交矩阵, 旋转
    * 等于其逆
    * TODO
- 对称矩阵
    * TODO



## 求解





