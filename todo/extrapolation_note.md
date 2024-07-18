---
title: 外推扫盲
author: 66RING
date: 2023-09-06
tags: 
- machine learning
mathjax: true
---

# 外推扫盲

> 训练时用的一部分数据, 可以训练到对这些特征的识别。当新的特征到来时, 模型要有识别这些新特征的能力(外推)。

- 直接外推: 训练时提前预留
    * 原始处理三维数据, 但训练时用四维训了(第四维空), 之后来四维数据时也能处理
- 线性内插: 外推改内插, 压缩到低维度
    * e.g. 0~2000压缩到0~1000, 相邻距离变拥挤
- 进制转换: 不加入新维度又不改变相邻距离
    * e.g. 10进制变16进制
- NTK外推
    * 类似RoPE, NTK会改变寻转的速度
- YaRN


- Ref
    * https://mp.weixin.qq.com/s/grayNg0IvAmILTF1dCEWTA
