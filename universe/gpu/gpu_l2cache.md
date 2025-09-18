---
title: GPU L2 Cache优化
author: 66RING
date: 2025-09-18
tags: 
- gpu
- hpc
mathjax: true
---

# GPU L2 Cache优化

In short: 总体需要经手L2 Cache的数据少了，命中率就高了

> https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html#l2-cache-optimizations
>
> L2 Cache是所有SM共享的cache

以gemm为例，假设一次最多调度4个thread block，用于计算C矩阵。

不优化的情况,  为了计算如下的C矩阵, 需要读取A, B矩阵总共5块。

```
   B B B B
A  0 1 2 3
   x x x x
         C
```

grouped优化(block id reorder)后, 只需要读取A, B矩阵总共4块。

```
   B B
A  0 1 x x
A  2 3 x x
         C
```

依旧是熟悉的Z字形排布
