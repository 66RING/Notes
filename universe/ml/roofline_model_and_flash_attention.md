---
title: Roofline Model and Flash Attention
author: 66RING
date: 2025-08-06
tags: 
- hpc
- machine learnring
- llm
mathjax: true
---

# Roofline Model and Flash Attention

- 计算强度: Flops/Byte => 反映每个字节的传输会产生多少计算
- roofline model:
    * 纵轴: 算力, 单位Flop/s
    * 横轴: 计算强度I, 单位Flop/Byte
    * 斜率: 内存带宽, 单位Byte/s
    * 算力的上限决定了roofline model的屋顶
    * 带宽的上限决定了roofline model的屋檐斜率
    * 拐点左侧: mem-bound, 因为单位byte传输没能用满算力
        + 理论上**平均**一个Byte的传输带来F/B次的mm就能打满
    * 拐点右侧: compute-bound, 算力已经打满了
        + e.g. 你的算法在某个规模上的计算强度I > F_peak/B

## 矩阵乘法的计算强度

mm的flop数F = 2xMxNxK, 传输数B = 2(bf16)x(MxK + NxK)

平均计算强度I = F/B = (MxN)/(MxK + NxK), 计算强度随着M和N的增大而增大, 理论上就可以打满算力

## Flash Attention的计算强度

设序列长度S, 维度数D, batch数B

- flop
    * QK的flop = 2xBxS^2xD
    * SV的flop = 2xBxS^2xD
    * softmax的flop = 3xBxS^2
        + 3表示softmax的exp, sum, div
        + 当D>>1时, softmax的flop可以忽略不计
    * 整体flop = 4xBxS^2xD
- 传输
    * Q的传输 = 2(bf16)xBxSxD
    * K的传输 = 2(bf16)xBxSxD
    * V的传输 = 2(bf16)xBxSxD
    * O的写回 = 2(bf16)xBxSxD
    * 整体传输 = 8(bf16)xBxSxD

A100的算力(bf16)FLOP/S = 312TFlops, 带宽为2TB/s, 计算强度拐点 = 312/2 = 156

Flash attention的计算强度I = F/B = (4xBxS^2xD)/(8(bf16)xBxSxD) = S/2

所以当S > 312时, Flash Attention就会打满A100的算力, 进入compute bound




