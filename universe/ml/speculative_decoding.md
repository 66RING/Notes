---
title: 投机采样推理原理
author: 66RING
date: 2024-03-11
tags: 
- machine learning
- inference
mathjax: true
---

# 投机采样推理原理

- 基本原理
- 正确性证明
- 推理加速证明
- 源码实现


## 基本原理

一次正确的生成其实使用小模型模型就能**大概率**正确的完成, 但连续多次的正确的概率就会下降, 一步错步步错。如果以大模型为基准, 则大模型的生成的百分百正确的, 但是大模型相对小模型需要耗费更多的时间来完成。

在自回归模型中, 由于因果关系的存在, 模型在推理时每次只能生成下一个token, 这种一次生成一个token的阶段在推理中称为decoding。这种一次生成一个的自回归机制极大的影响了执行的并行性, 如果大模型能够知道"未来的token", 那就可以提前计算下一个token。当知道token n+1时就可以计算token n+2, 知道token n+2时就可以计算token n+3。因此, 如果大模型预先知道"未来的token"的可能值n+1, n+2, ...那就可并行地执行n+2, n+3, n+4的推理。

基于此, speculative decoding的原理就是先使用一个小模型快速的生成未来可能出现的token: n+1, n+2, n+3... 然后利用大模型并行推理来验证正确性。如下图所示

1. 先使用小模型执行N(这里取5次)此自回归推理, 得到的结果token id分别为：289, 9, 39, 42, 20
2. 然后将小模式的预测结果拼接为大模型的输入执行一次推理, 因为大模型执行推理的过程中会得到中间token的预测值
3. 如果小模型的生成的token和大模型生成的token一致则可以取用该token, 当一个token错误时后续的token都不能使用, 即使后续存在正确token

![](https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/ml/speculative_decoding.png)


## 加速证明

以上图为例, 单纯使用大模型推理时生成5个token需要5次推理。而speculative decoding需要5次小模型推理和一次大模型推理。因为小模型的执行要比大模型快, 所以即使speculative decoding执行了一共6次推理, 时间还是划得来的。

假设小模型一次推理时间为t, 大模型一次推理时间为2t, **如果小模型结果完成正确**

- 原始推理时间 = 5 x 2t = 10t
- speculative decoding = 5 x t + 1 x 2t = 7t

当然实际情况是小模型完全正确的概率很低, 所以可以通过调整max token等超参来优化


## 正确性证明

以上图为例, 大模型推理结果为: `324, 56, 123, 80, 78 | 289, 9, 16, 42, 20`, 其中289为`324, 56, 123, 80, 78`的推理结果。`9`为`324, 56, 123, 80, 78, 289`的推理结果。如果289这个结果正确则9这个结果就可以使用。


## 源码实现

> [transformers assisted_decoding](https://github.com/huggingface/transformers/blob/4b796978656e461177a83d58ec3c2b06152c63db/src/transformers/generation/utils.py#L4186)

看hugging face的源码实现, 流程大致如下:

1. 使用小模型(`assistant_model`)推理N次
    ```python
    for _ in range(int(assistant_model.max_assistant_tokens)):
        assistant_model_outputs = assistant_model()
    new_token = assistant_model_outputs.logits[:, -1, :].argmax(dim=-1)
    ```
2. 拼接小模型的结果作为大模型的输入
    ```python
    candidate_input_ids = torch.cat((candidate_input_ids, new_token[:, None]), dim=-1)
    outputs = self(candidate_input_ids) # model()
    new_logits = outputs.logits[:, -candidate_length - 1 :
    selected_tokens = new_logits[:, -candidate_length - 1 :, :].argmax(dim=-1)
    ```
3. 用大模型的结果验证小模型的结果, 选取正确的token
    ```python
    n_matches = ((~(candidate_new_tokens == selected_tokens[:, :-1])).cumsum(dim=-1) < 1).sum()
    valid_tokens = selected_tokens[:, : n_matches + 1]
    ```

## ref

- https://github.com/hao-ai-lab/LookaheadDecoding
- https://zhuanlan.zhihu.com/p/653935025


