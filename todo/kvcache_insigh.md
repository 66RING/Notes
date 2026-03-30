---
title: KV cache原理和实现
author: 66RING
date: 2024-01-22
tags: 
- machine learning
- machine learning system
- llm
mathjax: true
---

# KV cache原理和实现

- KV cache对内存的节省
    - 省wk, wv计算
        - 每次embedding新加入的k, v就可以了, 历史的embedding已经缓存好了

- 省attention计算? 答案是不省的
    * 因为在推理中, 自回归每次只会查询一个q, 而这个q需要和其他所有kv计算权重
    * 并不涉及对历史q的重新计算



