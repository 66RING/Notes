---
title: Cutlass In Short
author: 66RING
date: 2025-08-22
tags: 
- cuda
- hpc
mathjax: true
---

# Cutlass通俗理解

## 核心目的

> 核心目标: 消除矩阵乘法中对A, B矩阵的重复读取

- 核心: 内积变外积 -> 每个数据只会读第一次, 避免对A/B矩阵的重复读取
    - **最内层外积, 可复用的地方也用外积**, 体现就是MMA的warp level op

内积的问题: 如果每个thread负责C矩阵的一个元素C[m, n], 并且用下面这种方式循环(内积), 可以发现A, B矩阵会被重复读取。

```python
# 内积的问题
# 背景:
#   我们希望每个thread负责一个c矩阵元素的计算
for m in range(M):
    for n in range(N):
        # 每个thread负责一个[m, n]
        for k in range(K):
            C[m, n] += A[m, k] * B[k, n]
# WARN: 不同thread会读取同样的数据 -> A, B矩阵会被重复读取
#  e.g. t[0, 1], t[0, 2]都会读取M[0]这行
```

外积的情况: 把k循环提到最外层, **那thread的任务要如何排布? -> MMA**

**利用MMA, 我们可以: 多个thread负责不同A/B矩阵元素的加载, 同时算同时算C的不同位置**

```
for k in range(K):
    for m in range(M):
        for n in range(N):
            C[m, n] += A[m, k] * B[k, n]
```

没有MMA的年代是怎么操作的? **我们只能优化gmem -> smem的读取了, 一次协作式加载到smem, 然后用内积方法做smem数据的矩阵乘**


## cutlass tiling

1. Thread block tile: C矩阵分块, **每个块C[Mb, Nb]分给不同的thread block**
2. 一个C分块需要读取A, B矩阵的一个长条
3. warp tile: MMA, smem后C的分块, **每个warp负责C[mb, nb]的一个小块**
4. thread tile: MMA内部电路交换, 一般不使用thread level的操作, 外积真正起作用的地方

```cuda
for (int mb = 0; mb < M; mb += Mtile) {
    for (int nb = 0; nb < N; nb += Ntile) {
        // NOTE:
        //  1. 每个thread block负责C[mb, nb]分块
        //  2. thread协作, 加载A, B矩阵的一个长条
        //      sA = A[mb, k], sB = B[k, nb]
        for (int kb = 0; kb < K; kb += Ktile) {
            // NOTE: 进入到warp tile
            C[mb, nb] += WarpTileOp(sA, sB, C[mb, nb]);
        }
    }
}
```

WarpTileOp展开

```cuda
for (int m = 0; m < Mtile; m += warp_m) {
    for (int n = 0; n < Ntile; n += warp_n) {
        for (int k = 0; k < Ktile; k += warp_k) {
            // NOTE:
            //  1. 加载warp内A, B矩阵到寄存器
            //  2. MMA
            // NOTE: warp级别的算子, **外积真正应用的地方**
            fragC[m, n] += MMA(fragA, fragB, fragC[m, n])
        }
    }
}
```



# misc


- thread block tile
- warp tile
- ...
- predicate: 避免使用ifelse等跳转指令, 对硬件流水线不友好
- QA
    * 为什么不能直接从寄存器写到global memory
        + tensor core的设计: 每个thread拥有的结果在逻辑上是比连续的, e.g. 
            + v0, v1连续, v2, v3连续，但是(v0, v1)和(v2, v3)之间不连续
            + **直接写回一个thread每次只能写回两个结果, 但是每个thread每次最多是可以写回128bit的数据的**
            + **使用smem来整理数据**

## TODO:

- pipeline
- swizzle


