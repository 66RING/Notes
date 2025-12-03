---
title: CUDA占用优化
author: 66RING
date: 2025-11-26
tags: 
- hpc
- gpu
mathjax: true
---

# CUDA占用优化

> 二次吸收: https://medium.com/@manisharadwad/unlocking-gpu-potential-understanding-and-optimizing-cuda-occupancy-2f43ee01ad7e

优化问题, 由于分配的粒度问题(一个block一个block分配资源, 固定会有n_thread * reg, n_thread的整块占用), 存在内碎片会导致资源利用率问题

```
Per SM:
1. num_threads_per_block * num_blocks < max_threads
    thread数不超上限
2. num_blocks < max_blocks
    block数不超上限
3. num_reg_per_thread * num_threads_per_block * n_blocks < max_register
    寄存器数不超上限
4. num_smem_per_thread * num_threads_per_block * n_blocks < max_smem
    smem数不超上限
```

优化目标: 正好用满max_threads和max_register和max_smem(优先级顺序相同)

可以用`cudaOccupancyXXX`系列API做计算. e.g.

- `at::cuda::getCurrentDeviceProperties()->multiProcessorCount`获取SM数
- `cudaOccupancyMaxActiveBlocksPerMultiprocessor(func, thread_per_block)`获取最大能调度的block数
- ...


## SM

> thread数和register数用满就可以看作100%占用。block可视为thread的逻辑分组

规格参数:

- Thread slot
    * 一个sm最大可运行的thread数(和运行时block数无关)
- Thread block slot
    * 一个sm最大可运行的thread block数(和运行时thread数无关)
- Register
    * 所有可用寄存器数(和运行时thread数无关, 运行时均分)
- Shared Memory
    * smem也是限制的一环 

**优化目标**: 同时占满thread slot, thread block slot, register。但往往会因为某个指标用满导致其他无法用满。


假设一个硬件的SM规格如下

- thread slot = 2048
- thread block slot = 32
- register: 65536
- wave: AKA几个thread block是一个wave
    * 一个sm中可以并行执行的一组thread block
    * wave大小(固定): 一个SM中可以并行运行的thread block数
- waves per SM: AKA几个wave是一个waves
    * 运行时平分的任务规模
    * e.g. 运行时2640个thread block, 硬件资源: wave大小是2, 132个sm
        + waves per SM = 2640 / 132 / 2 = 5

case study:

1. thread和block没法同时占满
    - e.g. 运行时规定每个block有1024个thread。结果只用了2/32个block, thread就用满了
2. 不整除的碎片问题
    - e.g. 运行时规定每个block有768个thread。结果用了两个block, 还剩600个空闲thread空闲
3. 寄存器压力
    - e.g. 每个thread会用到40个reg, 结果40 x 1638 ≈ 65536用满了寄存器但是没用满thread
    - e.g. 运行时32个寄存器, 512个thread
        * 4 * 512 -> 用满所有thread
        * 32 * 2048 = 65536, 没有超过寄存器限制
        * **此时加了个局部变量, 寄存器数增加到33**
            + 无法调度4个block(因为超出寄存器限制)
            + 只能调度3个block => 3 * 512 = 1536个thread, 512 * 33 = 16896个reg
            + 严重空闲



## 优化方法

1. 只能选择block size(thread数)
    - warp的倍数, thread占满但不受block数限制
    - e.g. 128, 256, 512
2. 寄存器占用
    - 注意局部变量的使用
    - `-Xptxas -v`查看实际寄存器数
    - 使用`__launch_bounds__(MAX_THREADS_PER_BLOCK, MIN_BLOCKS_PER_SM)`提示编译器的寄存器申请
    - **手动unroll控制寄存器数量**





