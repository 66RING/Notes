---
title: HPC Profile
author: 66RING
date: 2025-03-11
tags: 
- misc
- gpu
- hpa
mathjax: true
---

# 算子性能预估

- 算力
    * FLOP/s
    * 拼尽全力每秒完成的浮点运算次数
- 带宽
    * byte/s
    * 拼尽全力每秒完成的内存交换量
- 计算强度Arithmetic Intensity(**计算访存比**)
    * FLOP/byte
    * 平均读入数据能用上多少运算
    * 理解角度
        1. 把BW用满能发挥出的算力
        2. "一次IO"的计算强度(强度一定有个"计量时间", byte size就是这里的时间)
- roofline model: **纵轴是算力, 横轴是计算强度**
    * 算力决定屋顶的高度, 带宽决定屋檐的斜率
    * 计算强度增加算力增加说明说明不少算力的极限而是带宽的极限: mem bound
    * 计算强度增加算力不变(没有达到相当的FLOP)说明算力的极限: compute bound
- 模型理论性能: 模型在计算**平台上**的算力
- 计算强度上限: I = 算力/带宽
    * 计算**平台上**单位内存交换最多用来进行多少次计算
    * aka 带宽跟上算力所需数据量


## 理论性能预测

1. 估计计算强度
2. 估计设备的计算强度: TFLOP, Bandwidth
3. roofline model对比计算强度得出bound原因
4. e.g. gemm的计算强度MNK([M,N] = [M,K]@[N,K])
    - 简单的三重循环的实现方式, k循环。**因为加法操作较多，需要考虑加法的影响**
    - FLOP: 2 * M * N * K, 三重循环, 考虑加法
    - MEM读写: MxN, NxK, MxN三个矩阵的读写
    - 而A100的FP16峰值算力是312TFLOP, 带宽峰值是2TB/s, 计算强度=156

## 实际性能预测

tips:

1. 实际带宽可能用不满, 并且主要受到IO影响。所以可以将读入读出时间左右预估时间，然后使用nsight工具测试进行性能分析。
2. 一次GPU计算可能不是一次op, 而是一次MMA。用不满实际计算强度的利用率就低
3. ncu workload分析: 带宽利用率


## tips

ncu

- Long Scoreboard Stall的本质原因是需要从Global Memory中访问的数据并未就绪，导致依赖这些数据的后续指令都无法执行
- SM Active Cycles: 可以看到SM是否负载均衡(max, min区别大)
















