---
title: cutlass cute compose语义级理解
author: 66RING
date: 2025-12-21
tags:
- hpc
- gpu
- cute
mathjax: true
---

# cutlass cute compose语义级理解

layout是一个映射, 可以将逻辑`(m, n)`映射到物理的index: `layout(m, n) -> idx`。当两个layout复合时, 该idx除了有"物理idx"的还有外, 还有一层"逻辑crd"的语义。如`A o B = A(B(i))`

1. i传入到B, 先转变为B视角的逻辑crd
2. crd经过B映射后得到一个新的idx'
3. 该idx'传入A时又得到A视角的逻辑crd'
4. 通过逻辑crd'访问A得到最终复合的index

所以, 当需要(或者以后需要)复合的情况下。一个layout的语义就可以看成逻辑视图到逻辑视图的映射`(m, n) -> (m', n')`。其中`(m', n')`是针对要复合的layout来说的。

- C(i) = A o B = A(B(i))
    * **输出的形状和B的形状相同**, 但是stride发生改变

case study: 从tv layout到物理地址: `(t, v) -> (m, n) -> index`

1. tv layout(tid, vid) -> (m, n)能够获取到一个逻辑坐标(m, n)
2. 使用改坐标(crd, coord)去访问tensor(也是一种layout), 能够得到物理地址
3. `tensor.compose(tv_layout)`
    - `(tid, vid) -> (m, n) -> index`

同理swizzle: `(m, n) -> (m', n') -> index`
