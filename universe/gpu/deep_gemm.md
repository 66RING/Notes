---
title: Deep GEMM解读
author: 66RING
date: 2025-03-18
tags: 
- hpc
- machine learning
mathjax: true
---

# Deep GEMM解读

## terms

- `_ss`, `_rs`表示A/B矩阵的位置
- math warp
    * 用于计算的warp
- data warp
    * 用于传输数据的warp
- st/ld
    * store, load, 写回, 读出
- barrier
    * 同步信号
- fence
    * 防止乱序, 指令屏障

- components
    * wgmma
        + 更大粒度的mma, warpgroup-level, 单次规模更大复用更多访存
    * ldmatrix/stmatrix
        + 向量ld/st指令
    * tma copy
        + 异步多维拷贝
    * tensor map
        + 任务映射，地址映射
    * scheduler
        + 物理地址/逻辑地址管理, 任务(迭代id)管理
    * 各种指令
    * ...


- fp8_gemm.cuh
    * gemm主体
- mma_utils.cuh
    * mma原子操作类/helper函数
- scheduler.cuh
    * scheduler类
- tma_utils.cuh
    * tma helper函数
    * swizzle
- utils.cuh
    * 其他helper

## scheduler

- 逻辑块物理块映射管理
- **迭代id转物理块id**, (抽象)


## mma

> wgmma: https://docs.nvidia.com/cuda/parallel-thread-execution/index.html?highlight=wgmma%2520mma_async%2520sync%2520aligned%2520m64n16k32%2520f32%2520e4m3%2520e4m3#asynchronous-warpgroup-level-matrix-multiply-accumulate-instructions

- quick start
    * 矩阵A可以在寄存器或者smem, 矩阵B必须在smem, smem中的数据必须16 byte对齐
    * 然后就是ptx指令和约定格式
- **wgmma: 更大粒度做协同, warpgroup一组做协同计算, 能更多复用访存**
    * thread -> (warp-level)mma -> (warpgroup-level) wgmma
- op dispatch(select)
    * 根据M, N的信息选择适合的原子op

## tma

> https://research.colfax-intl.com/tutorial-hopper-tma/

`cute::SM90_TMA_LOAD_2D::copy(desc_ptr, barrier_ptr, cache_hint, smem_ptr, crd_0, crd_1);`

TODO: review https://research.colfax-intl.com/tutorial-hopper-tma/


## fp8 gemm

详见注释: https://github.com/66RING/DeepGEMM/blob/main/deep_gemm/include/deep_gemm/fp8_gemm.cuh


- `fp8_gemm_kernel`: iter-k模式 + block gemm
    1. 分配stage空间，barrier
    2. data warp: 负载拷贝数据到smem, 生产者, threadIdx.x >= kNumMathThreads
        - 只用一个warp做拷贝: threadIdx.x == kNumMathThreads
        - iter-k
            * empty_barriers->wait(): 等待消费者
            * tma_copy(full_barriers): 发起tma, 拷贝A, B, A_scale
            * 单独额外处理不整除(不规则)的情况
    3. math warp: 负责使用拷贝好的smem数据做计算, 消费者, threadIdx.x < kNumMathThreads
        - 加载B_scale, 不全让data warp做拷贝，考虑一点负载均衡
        - iter-k
            * 读取smem中的scale_b, 还是负载均衡的考虑, 因为scale_b的math warp加载的, 先读scale_b
            * full_barriers->wait()等待data warp拷贝完毕
            * 读取scale_a, fence(准备发起wgmma)
            * **直接利用smem中的数据发起wgmma**, wgmma的AB矩阵可以在smem中
            * 通知下一次tma加载，empty_barriers->arrive(), (smem可以覆盖了)
            * fp8 scale: 注意warp-group level计算，用cuda core做scale时区分warp-group



