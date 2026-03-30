---
title: Megatron-LM中的流水线并行
author: 66RING
date: datetime
tags: 
- tags
mathjax: true
---

# Abstract

> schedules.py

## 1F1B without interleave

1. 单独计算warmup的情况(及流水线还没填满)
    - 流水线总stage数 - 当前rank
2. 预备, 获取接收端发送端tensor信息

TODO: bert为例, `pretrain_bert.py::forward_step`

- terms
    * microbatch, 图表中的横过去的1234标志

### pytorch DDP

- `torch.distributed.get_world_size()`
    * [pytorch分布式训练](https://zhuanlan.zhihu.com/p/76638962)

## 1F1B with interleave

# Preface


# Overview
