---
title: llama2.c体验
author: 66RING
date: 2023-12-08
tags: 
- machine learning
- machine learning system
mathjax: true
---

# llama2.c体验

> Have you ever wanted to inference a baby [Llama 2](https://ai.meta.com/llama/) model in pure C? No? Well, now you can!

为什么要体验llama2.c? 因为我现在在做一些大模型相关的东西, 但是设备资源又不是十分充足, 想要先通过一个很小很小的llama来验证跑通再迁移到实验环境。偶然发现llama2.c的环境很适合我的需求, 使用的模型为[tinyllamas](https://huggingface.co/karpathy/tinyllamas), 数据集是[TinyStories](https://huggingface.co/datasets/roneneldan/TinyStories)。另一方面我对底层比较感兴趣, 想看看他用c是怎么写的。顺便学习一下标准化的ml工程知识。

## 简单使用

使用llama2.c从0训练一个将儿童故事的模型:

下载llama2.c

```bash
git clone https://github.com/karpathy/llama2.c
```

下模型和数据集等所需数据

```bash
python tinystories.py download
python tinystories.py pretokenize
```

开始训练

```bash
python train.py
```

自由有限的情况可以修改train.py脚本中的超参, 包括用cpu训练, 如batch size:

```python
batch_size = 128  # if gradient_accumulation_steps > 1, this is the micro-batch size
max_seq_len = 256
vocab_source = "llama2" # llama2|custom; use Lllama 2 vocab from Meta, or custom trained
vocab_size = 32000 # the Llama 2 tokenizer has 32K tokens
# model
dim = 288
n_layers = 6
n_heads = 6
n_kv_heads = 6
multiple_of = 32
dropout = 0.0
```

将模型转换成huggingface格式

```bash
python export.py tinyllama-15m --checkpoint ./stories15M.pt --version -1
```

## 总结

- 流程
    1. 环境准备和提供方便的小工具, 封装到一个小脚本中, 如`tinystories.py`
        - python实现一个简易的cli工具
        - 模型转换hf格式转换等
    2. 预分词: 单独先分词是必要的, 因为有时候分词真的很费时间
    3. 训练


## 实现

## plan(可深入学习的东西)

- 怎么做分词: 什么encode, decode, eos id, bos id等
- 实现自己的一个llama2推理(llama2.rs)


### 预分词

1. 处理每个样本(常用map)
    - 取样本`text = example["story"]`
2. 对样本进行加工(e.g. sft)
3. 分词(tokenize)
4. 写回分词后的结果(下次可以直接`load_from_disk`加载分词好的数据)

### 训练

- 分词器训练
- 模型训练

### 推理

run.c






