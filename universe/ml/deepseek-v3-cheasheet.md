---
title: DeepSeek-V3架构
author: 66RING
date: 2024-12-02
tags: 
- machine learning
mathjax: true
---

# DeepSeek-V3架构

## MTP(Multi Token Prediction)

TODO:

## RL 强化学习

- 对于数学, code等有明确答案的，直接使用规则做reward
- 对于自由形式没有明确答案的任务，使用reward model提供反馈
- 如果是写作等没有明确结果的，使用reward model打分
- 奖励会分成多步基于，而不只是检查最终结果

- GRPO
    * vs PPO
    * TODO

## 混合精度

> 把gemm拆分成mul和sum, 在利用量化和低精度mul加速计算，使用高精度sum防止溢出

矩阵乘法的精度溢出问题: gemm = sum(mul), sum过程容易产生溢出，尤其是维度较大的。可以mul时在tensor core上低精度, sum时在cuda core上高精度进行。

为了加速和用满乘法单元, 可以分块进行从而凑齐一个block的大小。

FP8量化, 每个block上再做FP8量化即可。反量化则是乘上一个scale因子(当然可能还有offset)。

所以, 量化 + 低精度做mul, 再高精度做sum, 从而加速。

1. 利用量化和low bit加速计算: 更少的cycle更多的硬件
2. 使用高精度sum防止溢出

e.g. 

```
C = A @ B
4分块
C11 = A_b1 * B_b1 + A_b2 * B_b2 + A_b3 * B_b3 + A_b4 * B_b4
量化
C11 = A_qb1 * B_qb1 + A_qb2 * B_qb2  + A_qb3 * B_qb3  + A_qb4 * B_qb4
反量化
C11 = A_qb1 * B_qb1 * A_dq1 * B_dq1 + A_qb2 * B_qb2 * A_dq2 * B_dq2 + A_qb3 * B_qb3 * A_dq3 * B_dq3 + A_qb4 * B_qb4 * A_dq4 * B_dq4

所以总的流程belike
1. low bit + quant mul + dequant
block_mul = [
    A_qb1 * B_qb1 * A_dq1 * B_dq1,
    A_qb2 * B_qb2 * A_dq2 * B_dq2,
    A_qb3 * B_qb3 * A_dq3 * B_dq3,
    A_qb4 * B_qb4 * A_dq4 * B_dq4,
]
2. high bit sum
C11 = sum(block_mul.to(fp32)) # kernel实现一般就是一个fp32的累加器+=cast(x)
```

## DualPipe

> 看图: 横坐标时间轴。纵坐标compute communication标志计算单元在做什么通信单元在做什么。相同颜色表示一个流程, B+W组成完成的反向过程

- workload
    * fwd
        1. F: 计算激活值
    * bwd
        1. B: 梯度链式传递
        2. W: 更新权重

所以zero pipe: 前向反向重叠1F计算后分发通信的同时计算1B, 1B完后本地更新权重1W同时进行1F的通信分发, **即, 1B后就可以发起通信, 不需要BW完才发起通信，从而减少气泡**。

- MoE flow
    1. attn
    2. dispatch (to moe)
    3. mlp (moe)
        - 会计算topk个专家的MLP: `[B*L, topk, D]`
    4. combine
        - topk个专家的结果加权求和`sum([B*L, topk, D] * [-1, topk, D], dim=1)`
    5. attn
    6. ...
- dualpipe
    * 每张卡上都会存在两个数据并行, 从而实现了前向和反向的重叠计算(一个前向时一个反向)
    * 一批数据从device0走到device7, 另一批数据从device7走到device0。既一个device上有两个layer的数据
    * **相当于chimera(奇美拉) + zero pipe**



