---
title: GRPO Cheat Sheet
author: 66RING
date: 2025-09-25
tags: 
- machine learning
- reinforcement learning
mathjax: true
---

# grpo cheat sheet

> https://huggingface.co/docs/trl/main/grpo_trainer

<img src="https://huggingface.co/datasets/trl-lib/documentation-images/resolve/main/grpo_visual.png" alt="">

GRPO(Group Relative Policy Optimization)的4个步骤: 其中1,2阶段相当于准备阶段, 3,4阶段相当真正的训练阶段

1. 生成补完(Generating completions)
    - AKA推理生成, **不带梯度**
    - 同一个输入生成多个补全(**group**)
2. 计算优势(Computing the advantage)
    - AKA用reward model/function对所有回答进行打分, **不带梯度**
    - 可以选择对不同的reward进行加权
3. 计算loss, 准备阶段结束
    - 估计KL散度(计算loss的一部分)
        - 计算策略模型和参考模型的KL散度, **带梯度**
    - 公式(优势, kl散度), **带梯度**
4. 反向传播


GRPO的组件

- 策略模型(Policy model)
    - 用于生成回答
- [Optional]参考模型(Reference model)
    - 用于和策略模型的输出进行对比(计算KL散度)
- 奖励模型(Reward model)
    - 用于对回答进行打分, 模型或者函数

数据流:

```python
def train_step():
    model.eval()
    # 1. completion
    input = ["prompt"] * group_num
    # take vllm as an example
    policy_model_output = vllm_inference(input)  # no grad
    refer_model_output = vllm_inference(input)  # no grad
    # 2. compute reward(advantage)
    rewards = [r(policy_model_output) for r in reward_func_list] # no grad
    # [Optional] reweight reward
    rewards = rewards * reward_reweight

    # 3. compute loss, start traint
    # NOTE: grad required
    model.train()
    kl = compute_kl(policy_model_output, refer_model_output)
    loss = compute_loss(kl, rewards)
    loss.backward()
```
