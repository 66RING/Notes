---
title: 大模型的长序列优化survey
author: 66RING
date: 2023-08-28
tags: 
- machine learning
- machine learning system
mathjax: true
---

# 大模型的长序列优化

为什么长序列是必要的? 

- prompt engineering
- 会话, 书籍, 视频
- 不忘记之前说的话, 不分心

- idea
    * prompt hashing!
        + 允许精度损失, 浮点数类似的精度设计
    * 如果一个token表示的不是一个token, 而是一个区域? 再加一个encode向量数据库
    * 相关图, k邻近
    * 多模态有没有可能就是将其他各种信息都embading进去

- 记忆力和遗忘曲线
    * page rank, 外存, (rank, relative)偏序


## LongNet

> [LongNet](https://arxiv.org/pdf/2307.02486.pdf)
>
> [code](https://github.com/microsoft/torchscale)

- overview(zhihu)
    * 复杂度从二次降低到线性
    * 扩大序列长度的主要挑战是在计算复杂性和模型表达能力之间取得适当的平衡
    * "扩展注意力"变体
    * 序列切分, 分配到不同设备上
    * 计算复杂度和模型表达能力
- 扩展Attention机制(Dilated Attention)
- 优势
    * 线性的计算复杂度
    * 可做非常长序列的分布式训练器
    * 无缝集成到transformer中

本质就是Sparse attention, 只是选取的pattern比较讲究

- observation
    * token的一个一个无法并行
- related work
    * CNN, RNN等结合transformer, 但是这会忘掉早期的prompt
        + 注意力前对input做卷积缩放窗口
    * 稀疏的attention
- intro
    * Sparse attention:
        + 只有key, value的子集会参与运算, 从而降低计算复杂度, 使用一个S矩阵(pattern)决定哪个会参与运算
    * Dilated Attention:
        + **等间隔取**QKV片段
        + 可以有多种取法, 这些取法就可以**并行传入attention**, 最后拼接输出
        + 为了同时兼顾长序列和短序列, 使用混合的Dilated Attention模式
- LongNet Advantage
    * attention allocation decreases exponentially as the distance between tokens grows
        + 线性复杂度
        + Distributed Algorithm, 可扩展性
            1. split input到各个设备中
            2. 投影(projected, 选出)QKV进行Dilated Attention计算
                - 因为pattern比较规律, 所以有比较好的可扩展性

- QA
    * 输入的并行 check
    * 生成的并行 TODO: KV cache?
    * 没有和其他对比准确度, 只说明了自己的方法确实可扩展和有效


- idea
    * Sparse attention很容易用页表实现, 且很容易节省内存和做内存管理
    * 不等间隔取: 类似浮点是的精度, 从而提升表达能力
        + 但怎么做到Dilated Attention那样的可扩展性? 线性+一点非线性?
        + 或者语法感知, 找到侧重点? 添加到可微调的内容
    * 二次遗忘(deja vu)
    * 先保证相关(过滤器), 再保证提取


## LongLlama

> [LongLlama](https://arxiv.org/pdf/2307.03170.pdf)

- 提出FoT(Focused Transformer)
- 上下文长度在256k时仍有73%的准确率, 而标准的LLaMA-3B在2K时准确率就0了

- overview
    * 受对比学习启发

kNN lookup to pick up the most relevant tokens

本质就是embad(prev, curr), 只是prev和curr的选取比较讲究。本身对架构没有太大变化, 优化输入的选择和注入一定的历史记忆。

- 论点和挑战
    * TODO:
- observation
    * 现有的并行系统(数据, 流水, 模型, 张量并行)无法支持高效的长序列训练
        + 序列维度扩展问题, 内存通信效率问题, 易用性和侵入性问题
    * 标准的训练在长序列场景下经常有相关和不相关token重叠的情况, 导致不知道该选哪个

- related work
    * 降低平方的计算复杂度
    * 通过微调在增加文本长度
    * 对比学习(Contrastive learning), 通过对比学习来区分好坏sample
- FoT(Focused Transformer)
    * Memory attention layers: enable the model to retrieve information from the external memory at inference time
        + 使用kNN找外存中的(k, v)来填充上下文: attn + d
        + 每次query都会在当前context的基础上根据kNN从外存选取top k的keys
    * Crossbatch training procedure: biases the model to learn (key, value) representations
        + bias就是'+'的这个过程, embad(prev, curr)
        + 需要positive和negative的东西, 以便后续能训练出区分能力?
        + 从当前上下文相关的内容(previous的, current的)选出positive的, 从和当前上下文无关的内容中选出d-1个negative的(d可调)

- idea
    * 可插拔的fetch, 和分页管理似乎更贴合
    * kNN是不是图计算更好?
    * **ml存在记忆力吗? 不要输入多少知道多少**
        + 所谓记忆是不是就是"直觉脑"的训练?
    * 动态的d是否可行? 思路类似浮点数精度问题



## DeepSpeed Ulysses

> [DeepSpeed Ulysses](https://zhuanlan.zhihu.com/p/652206513)

- 一种序列的分布式计算方法
- 序列并行的本质是用多卡来处理一个序列, 比如原本的数据量是sd, 序列长度乘其他数据的大小d。序列并行后每个GPU就只用处理sd/N了


## misc

工程设计 + 算法设计

- RETRO: [Improving language models by retrieving from trillions of tokens](https://arxiv.org/pdf/2112.04426.pdf)
    * RETRO模型利用从大型语料库中检索到的文档块，基于与前面标记的局部相似性来增强自回归语言模型
- [Memorizing Transformer](https://arxiv.org/abs/2203.08913)
    * kNN邻近, 检索最相关, 同样是local context + memory
        + 度信息不会回传到外部memory, 可以轻松的将memory扩展到比较长的token数，如262k长度
    * 方法:
        + decoder中间一层中加入external memory, 可以做fine tune
- [LoneMem](https://arxiv.org/pdf/2306.07174v1.pdf)
    * 看Figure 2就够了
    * TODO
    * 组成: LLM和SideNet解耦
        + 训练好的语言模型: 编码输入输出
            + **后期只用训练SideNet即可**
        + SideNet: 融合当前输入和之前输入, "作残差连接"
        + Cache Memory Bank(记忆缓存库): 存储之前模型的关键信息
            + AI数据库?
            + Serverless, 集体潜意识?
    * 实现
        1. 生成KV存到外存cache中
        2. 通过KV访问SideNet拿到Q
        3. 残差链接到网络中

- related work
    * 滑动窗口
    * 线性transformer
    * token池化
    * 稀疏attention
- idea
    * 做kNN时是不是也可以考虑ResNet? 即受输入反馈和更新
    * deepspeed virtual node? 数据全部换出GPU不就可以了
    * attention = 理性脑, memory = 直觉脑
    * 预测下一个词的抽象是不是欠妥, 应该是预测下一个模块, 然后模块内展开, 然后记忆可以回溯到high level再处理下一个模块
        + **AI懂得抽象吗?**
        + 抽象, 展开, 抽象, 展开
        + aka, 分模块的生成式AI
        + 参考[无限外推的ReRoPE](https://kexue.fm/archives/9708)中提出的窗口概念
            + 多层attention就相当于多层展开了
    * 外置大脑的多级调节: kNN, PageRank, trainable things
        + 滑动窗口
        + 外置大脑等价于细胞分化:
            + 原始脑: 本能需要, 速度块
            + 情绪脑: 社交支撑, 速度中
            + 理智脑: 进阶进化, 速度慢


## Ref

- https://blog.csdn.net/OneFlow_Official/article/details/130003013

