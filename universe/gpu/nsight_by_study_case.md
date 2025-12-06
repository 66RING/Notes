---
title: 使用Nsight分析GPU程序
author: 66RING
date: 2024-02-28
tags: 
- HPC
mathjax: true
---

# 使用Nsight分析GPU程序

https://www.bilibili.com/video/BV13w411o7cu/?vd_source=fa5227c06f0a0c9f870b406f10be1d31

# Cheat sheet

1. 占用: SM, Memory
    - mem-bound, compute-bound的特点是否符合
2. 看scheduler + 具体warp的怎么了(scheduler statistics)
    - 为什么Eligible和Issued的warp少
3. 看stall reason(Warp state statistics)
    - long scoreboard: 要读远距离的数据(e.g. global mem)
        - 优化:
            1. warp够多, 一个warp stall, 其他warp可以被调度
            2. 异步, 既然阻塞就不用立刻使用, 而是先干别的
    - LG Throttle: 读写gmem的**指令太多**
        - 优化:
            1. 合并访存
    - short scoreboard: smem和特殊指令
        - 优化:
            1. 异步, 既然阻塞就不用立刻使用, 而是先干别的
    - MIO Throttle: pipeline用得多, e.g. smem和bank conflict


- case study
    - https://developer.nvidia.com/blog/improving-gpu-performance-by-reducing-instruction-cache-misses-2/


# Sections

## SOL(Speed of Light)

- GPU利用率
    - compute bound => SM利用率高, memory利用率低
    - memory bound => SM利用率低, memory利用率高
    - latency bound => SM和memory利用率都不高
        - warp数写得不够多
- Breakdown(右上角切换视图)
    - 利用率是breakdown中metrics的最大值

## Compute Workload Analysis

> 添加参数`--section ComputeWorkloadAnalysis`, 或者直接`--set full`

各个单元的功能: https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#id28

- ADU, CBU
- ALU, FMA
    - 如果是compute bound, 应该是希望尽可能用在这些计算单元上

## Scheduler

> 为什么硬件单元没忙
>
> 没个周期的数据

- Active warps: 所有可以被调度发送的warps
- Eligible warps: 数据是准备好的warps
- Issued warps: 已经发射的warps

如果每个周期都能发射一条指令, 那希望Issued warps = 1 => 看Warp state分析

## Warp state

warp被阻塞的通常原因:

1. 正在获取指令
2. 等memory dependency
3. 等execution dependency
4. pipeline busy, e.g. 要用tensor core, 但是tensor core忙
5. sync barrier


stall reason

- stall long scoreboard
    - 从off-chip(global memory)的地方读取数据
- stall short scoreboard
    - 从距离相对短的地方(e.g. share memory, 特殊mfu, 动态分支等)读取数据
- LG Throttle
    - 等待L1 inst queue: 执行local/global memory指令太频繁, 阻塞
    - **指令队列导致的stall, 单条指令时间长(e.g. 读写gmem)**
- MIO Throttle
    - 被mem input/output指令阻塞, e.g. LDS, MUFU, 动态分支
    - **IO导致的stall**
- Stall Not Selected
    - 一个warp可发射, 但是选择了其他warp。说明warp数量很高，可以适当降低
- Stall Memory Throttle
    - 一个warp不可发射, 因为LSU pipe被占用。说明smem的使用有gap, 可能是bank conflict或warp divergence
- Stall No Instruction
    - means the SMs could not be fed instructions fast enough from memory.
    - instruction caches miss => 指令太多样了
        - **warp间的执行相互影响**
        - 区分prologue, main loop, epilogue，保证指令执行的相似性和局部性


## Memory workload

## Launch statistics




