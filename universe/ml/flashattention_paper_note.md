---
title: FlashAttention笔记
author: 66RING
date: 2023-08-19
tags: 
- machine learning
- machine learning system
mathjax: true
---

# FlashAttention 1

- main idea
    * IO感知, 即感知GPU的层级关系
    * 手动算子融合, 实现CUDA算子
- 局限和Future
    * 需要手写CUDA做融合, 希望可以用高级语言写在编程成CUDA
    * IO感知的思路可以扩展到非Attention的场景
    * 多GPU的IO感知也可以做优化
- 实现
    * 尽可能少设计HBM读写
        1. 计算softmax时不需要访问整个输入
            - 重新设计attn的计算, 让输入可以分块多次地计算: **tiling**
        2. 反向时不存储大量中间结果
            - 保存前向时softmax normalization factor以快速重算, 而不是传统方法的需要读取中间数据: **recomputation**
    * 具体实现: tiling, recomputation. [ref](https://www.zhihu.com/question/611236756/answer/3132304304)
        1. tiling: 分块加载分块计算。Q, K, V分块加载到SRAM, 分块单独计算
            - softmax公式转换, **关键在于如何通过局部值在最后换算出全局值**
                1. 分母直接用最新标量值, 分子部分要将指数位更新成全局值, e.g. $(\sum e^{x_i^{(2)} - m(x^{(2)})}) * e^{m(x^{(2)} - m(x_{new}))}$
                    - in short **相乘等于指数位相加** 从而替换上新值
        2. recomputation: 不存储方向传播需要的中间值
            - 通过存储softmax normalization statistics (m,l)和输出O就可以重计算S和P
        3. kernel融合

$$
softmax(x) = \frac{[f(x^{(1)}) \cdot e^{m(x^{(1)}) - m(x)} , f(x^{(2)}) \cdot e^{m(x^{(2)}) - m(x)}]}{\sum{[l^{new1}, l^{new2} ] }}
$$

# FlashAttention 2

PS: block具体大小应随GPU变化

> [ref](https://zhuanlan.zhihu.com/p/645376942)

- main idea
    * FlashAttention1还不是最优, 主要是因为任务的划分在不同GPU thread blocks, wraps下不是最优
        + 一个grid包含多个block, 一个block包含多个wrap, 一个wrap包含多个thread
    * 更好的work partitioning
        1. 减少非乘法(non-matmul)操作
        2. 并行计算attn, 即使是单头
        3. 考虑多在thread block内计算, 减少跨组通信
- discussion and future
    * 让flashAttention2兼容更多设备和数据类型
    * 利用编译器让编程更简单
- 实现
    * matmul优化
    * 并行化
        + 额外在序列长度这一维度考虑并行, 提高GPU利用率
            + 因为序列长度大时会降低batch size, 从而降低GPU利用率
    * wrap分配和循环调整 TODO
        + 调整公式从而跳转循环的实现, 结果是HBM的读写更少了
        + 内外循环调整: 原本KV在外循环, QO在内循环
            + 跳转后Q在外循环, KVO在内循环, 降低wrap之间的通信TODO







