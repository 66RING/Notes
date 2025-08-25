---
title: Flash Attention v3技术点
author: 66RING
date: 2025-08-24
tags: 
- gpu
- hpc
mathjax: true
---

# Flash Attention v3技术点

> hopper特性co-design

- hopper新特性
    * 低精度
        + 优化量化误差
    * 异步
        + intra-warpgroup, inter-warpgroup

## 异步

- 异步: 异步计算(WGMMA), 异步传输(TMA) => 软件级流水线
    * producer-consumer模型: 生产者传输, 消费者计算
    * 算力高的tensor-core(gemm)去覆盖算力低的cuda-core(softmax, exp)
    * inter-warpgroup overlap
        + gemm和softmax做overlap
    * intra-warpgroup overlap
        + gemm0算ntile + 1时, gemm1算ntile
            1. wait WGMMA0 complete
            2. do online softmax
            3. wait WGMMA1 complete

- warp-specialized
    * hopper:
        + warp的寄存器分配有区分, e.g. TMA只用少量寄存器
    * hopper之前:
        + 所有warp的寄存器分配一视同仁, 不区分，导致寄存器浪费


## 低精度

- 挑战
    1. per-block quant
    2. gemm-I的C和gemm-II的A矩阵layout不兼容


- per-block quant
    * C = A * B * scale_A * scale_B
- fp8场景下gemm融合的layout不兼容问题: gemm-I的C layout和gemm-II的A layout不兼容
    * 解决方案: cutlass3.5 => 构造tileMMA时指定permutationLayout
        + `make_tiled_mma`


### **permutationLayout**

> https://www.bilibili.com/video/BV1kkL7zrEMW/?spm_id_from=333.1387.upload.video_card.click&vd_source=fa5227c06f0a0c9f870b406f10be1d31

permutationLayout: 给你一个row-ptr, 你构造stride去重排这一行


``` python
"""
Layout<
  Shape<2, 4, 2>,
  Stride<1, 4, 2>,
>
"""
index = []
shapes = [2, 4, 2]
strides = [1, 4, 2]

shapes.reverse()
strides.reverse()
for i in range(shapes[0]):
    for j in range(shapes[1]):
        for k in range(shapes[2]):
            index.append(i * strides[0] + j * strides[1] + k * strides[2])
print(index)
```







