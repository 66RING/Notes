---
title: 如何将梯度下降算法变成分布式的梯度下降算法
author: 66RING
date: 2023-05-09
tags: 
- machine learning
- system
- distributed
mathjax: true
---

# 如何将梯度下降算法变成分布式的梯度下降算法

> [Scaling Distributed Machine Learning with the Parameter Server](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-li_mu.pdf)

- scheduler
    * 通知所有worker加载数据, `LoadData()`
    * 通知worker启动并分批处理小批量的数据, `WorkerIteration(t)`
- worker
    * LoadData
        1. 读取对应块的数据
        2. 从server处获取对应的weight
    * WorkerIteration(t), 第t个迭代(第t小批)
        1. 算小块梯度
        2. 梯度传回服务器
        3. 拉取下一批weight, 从server处获取对应的weight
- server
    * ServerIteration 
        1. 计算全局的梯度
        2. 更新weight


- 异步执行如何处理依赖性
    * 不同的依赖关系以提供不同的一致性模型: N-Bounded delay. 支持自定义调节, i+N必须等待i完成



