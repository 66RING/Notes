---
title: 自动并行insight, 关于alpa和unity的思考
author: 66RING
date: 2023-10-01
tags: 
- llm
- machine learning
mathjax: true
---

# 自动并行insight: 关于alpa和unity的思考

[Alpa](https://www.usenix.org/conference/osdi22/presentation/zheng-lianmin)和[Unity](https://www.usenix.org/conference/osdi22/presentation/unger)都是OSDI22上关于大模型自动并行的文章。居然戏剧性的同时入选，工作量当然是一个重要因素, 不过这个偶遇也给了我一些思考，浅浅记录一下。下一期会开"斯坦福自动并行三剑客源码解析: MetaFlow, TASO, FlexFlow(Unity)"。

首先会先简述一下Unity和Alpa的思路。会先说Unity再说Alpa, 这样对比会更强烈一些:)。

一句话总结就是: 

- Unity使用枚举子图替换的方式实现了所有情况的考虑, 最终能不断递归/迭代/展开从而考虑到所有的组合情况选出最好, 完全自动的做自动并行
- Alpa的自动并行先被人工拆分成了两个阶段, inter(e.g.流水线并行)和intra(e.g.张量并行), 这是符合工程经验的, 然后Alpa只需要"半自动"地处理怎么拆分流水线, 怎么拆分算子即可


## 什么是自动并行

首先什么是自动并行, 简单的说自动并行就是将算子(aka 运算符 aka 操作符), 计算过程自动并行化。怎么个并行化法? 在大模型中常用的并行方式有流水并行, 数据并行, 张量并行等。并行化的体现通过例子说明就是:

- 流水并行: 神经网络多层的layer分配到多卡上执行
    * 类比CPU流水线
- 数据并行: 
- 张量并行: 

上面这些并行方式基本就是自动并行要实现的目标(当然还有序列并行等其他并行)。

## Unity

> 枚举is all you need, 递归的模式替换

Unity最本质的实现(MetaFlow)就是: 子图**枚举**, 模式替换。整个进化过程: MetaFlow中添加自图替换机制, TASO中添加变换自动生成, 最后到FlexFlow(Unity)的分布式计算图(PCG)表示。

观察MetaFlow会发现, 他的子图替换的模式是真的人工设计并写死的。

![SubGraph replacer](https://picx.zhimg.com/80/v2-8a134d917ca665ae64cffc1817cb68a9_1440w.png)

然后到了TASO, 就可以利用一些数学计算的规则进行子图替换模式的自动生产，数学计算的规则也是人工写死，然后dfs枚举的。

![TASO](https://picx.zhimg.com/80/v2-9da52d78b876ff761dadeaeb0bfdc421_1440w.png)

最后到了Unity阶段, 他们抽象出一套分布式计算的表示, 这些分布式操作看成算子继续做子图替换就能实现自动并行。

![Unity](https://picx.zhimg.com/80/v2-e6517159bab540c369233a04f312543e_1440w.png)

如果说这些分布式算子是**成对出现**的话, 设为(PS, PE), 那么自动并行就可以是先用一对PCG算子将一计算子图包围, 从而实现向计算图中引入分布式算子, 然后在对子图进行展开和枚举就可以了。

看下面这个例子: 

1. partition和combine可以对出现, 通过在一个batch MM操作前后插入这对算子从而将分布式算子引入计算图
2. 然后就是正常的子图枚举和模式替换了
3. 如softmax调整到combine前就可以利用分布式计算提速了

![Unity demo](https://picx.zhimg.com/80/v2-519728e77e611f68b1da754e2b8ee6d2_1440w.png)

同MetaFlow, 这些分布式算子的插入也是人工写好的固定模式:

![pcg xfer](https://picx.zhimg.com/80/v2-89cd5d562120a089c3833f7b458d9e4f_1440w.png)

一句话总结就是: 枚举所有可能的等价变化(当然有上限阈值), 组合都尝试一下, 这些变化是人工生成或者自动生成的。


## Alpa

> 半自动并行

Unity的枚举很酷, 感觉只要给出足够了模式和足够的时间, 就能通过模式替换的枚举出所有计算图并选出最合适的。

但是其实你并不需要从0开始枚举, 因为我们目前知道的就几种并行方法, 无非就是流水并行, 数据并行, 张量并行和序列并行等。而且**大概率(或者)是先做数据并行, 再到流水并行, 最后到张量并行的**。所以Alpa就是一种"半自动并行"思想。

既然是先做流水并行的那Alpa就先用DP动态规划方法看看怎么分配流水线就好啦(inter parallel)。

![alpa inter parallel](https://picx.zhimg.com/80/v2-eba84baceedd8fbde74e62af98e50814_1440w.png)

流水并行后就是每个流水线Stage内的张量并行, 那我就可以用ILP线性规划看看算子怎么拆分就ok。

![alpa intre parallel](https://pica.zhimg.com/80/v2-8ad9661e52d7db2b1d7dfb1e5e59f01f_1440w.png)

因此, Alpa将自动并行的流程拆分成符合工程经验的先流水并行再张量并行, 从而实现了"半自动并行"的效果, Alpa虽然无法像Unity一样枚举到所有情况, 但是Alpa生成出的东西大概率在最优那个梯队里面。

