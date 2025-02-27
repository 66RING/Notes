---
title: 神经网络中的激活值
author: 66RING
date: 2024-01-27
tags:
- machine learning
mathjax: true
---

# 神经网络中的激活值

- 什么是激活值
    * 任意算子的输出结果就是激活值, 因为你需要使用结果(激活值)来求导
- 为什么要保存激活值
    * 因为反向传播过程中需要**对函数的参数求偏导**, 那么结果就和激活值相关
        + 如: f(x) = ax, g(x) = bx, 神经网络为: g(f(1))
            + $d = f(1)$, $g'_b(x) = \frac{dg}{db} = x$, $f'_a(x) = \frac{df}{da} = x$
            + $g'_b(x) = \frac{dg}{db} = d * f'_a(x) = d * x$, 其中d就是激活值
    * 激活值保存在pytorch中往往**体现为保存输入**: `ctx.save_for_backward(input)`
- 激活值显存占用的计算
    * 因为自动求导机制的存在, 激活值往往不再以layer为单位产出, 而是以算子(加减程序, relu, attn等)为单位产出
    * 和具体实现有关: 即`ctx.save_for_backward(input)`




