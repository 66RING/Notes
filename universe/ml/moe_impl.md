---
title: MoE基本实现
author: 66RING
date: 2024-12-03
tags: 
- machine learning
mathjax: true
---

# MoE基本实现

in short

1. topk expert选择
    - linear(dim, num expert)赋权
    - topk at dim -1
2. **permute**: 让同一个expert的token揍在一起, 方便做一组mlp(grouped gemm)
    - 技巧:
        1. `topk_ids.view(num_token, topk).view(-1).argsort()`会根据topk排序, 相同expert的token会被排在一起
        2. token-wise重排(permute): `sorted_tokens = hidden_states[indices // topk]`, idx / topk得知原本token的位置
        3. 每个专家的token长度记录: `(num_token, num_experts).sum(0) -> (num_experts)`每个专家的token数统计
        4. for循环每个专家做mlp


## code

> transformers ... modeling_deepseek_v2.py

```python
def self.gate(self, hidden_states: torch.Tensor):
    """
    x -> linear(dim, num_experts) -> topk(k) -> (num_tokens, topk)
    """
    batch_size, seq_len, hidden_dim = hidden_states.shape
    ### compute gating score
    hidden_states = hidden_states.view(-1, hidden_dim)
    logits = F.linear(hidden_states.type(torch.float32), self.weight.type(torch.float32), None)
    # shape = (num_tokens, num_experts)
    scores = logits.softmax(dim=-1, dtype=torch.float32)

    # select top-k experts
    topk_weight, topk_idx = torch.topk(scores, k=self.top_k, dim=-1, sorted=False)
    topk_weight = topk_weight * self.routed_scaling_factor
    return topk_idx, topk_weight

def moe(self, hidden_states: torch.Tensor, topk_ids: torch.Tensor, topk_weight: torch.Tensor) -> torch.Tensor:
    """
    Args:
        hidden_states shape = (num_tokens, dim)
        topk_ids shape = (num_tokens, topk)
        topk_weight shape = (num_tokens, topk)

    Returns:
        hidden_states
    """
    # NOTE: zeros with the same device and dtype
    # (num_tokens, num_experts)
    cnts = topk_ids.new_zeros((topk_ids.shape[0], len(self.experts)))
    # NOTE: onehot
    # (num_tokens, num_experts)
    cnts.scatter_(1, topk_ids, 1)
    # NOTE: (num_experts)
    tokens_per_expert = cnts.sum(dim=0)
    # NOTE: expert routing
    #  1. indices = token_id * topk
    #  2. 并且是根据topk排序的, 所以expert id靠前的会被排在前面
    indices = topk_ids.view(-1).argsort()
    # NOTE:
    # 1. permute token: e.g.
    #  expert0_token0
    #  expert0_token1
    #  expert0_token2
    #  expert1_token0
    #  ...
    sorted_tokens = hidden_states[indices // topk_ids.shape[1]]

    # Process experts
    outputs = []
    start_idx = 0
    for i, num_tokens in enumerate(tokens_per_expert):
        if num_tokens == 0:
            continue
        end_idx = start_idx + num_tokens
        expert = self.experts[i + self.ep_rank * self.experts_per_rank]
        # NOTE: 拿expert需要处理的token
        tokens_for_this_expert = sorted_tokens[start_idx:end_idx]
        # NOTE: mlp like: self.down_proj(self.act_fn(self.gate_proj(x)) * self.up_proj(x))
        expert_out = expert(tokens_for_this_expert)
        outputs.append(expert_out)
        start_idx = end_idx

    outs = torch.cat(outputs, dim=0) if outputs else sorted_tokens.new_empty(0)

    # Reorder and combine outputs
    new_x = torch.empty_like(outs)
    # NOTE: 插入对应位置
    new_x[indices] = outs
    hidden_states = (
        new_x.view(*topk_ids.shape, -1)
        .type(topk_weight.dtype)
        .mul_(topk_weight.unsqueeze(dim=-1))
        .sum(dim=1)
        .type(new_x.dtype)
    )
    return hidden_states

def forward(self, hidden_states: torch.Tensor) -> torch.Tensor:
    """ hidden_states shape (bsz, seqlen, dim)
    """
    residuals = hidden_states
    orig_shape = hidden_states.shape
    # NOTE: (num_tokens, topk)
    topk_indices, topk_weights = self.gate(hidden_states)
    # NOTE: (num_tokens, dim)
    hidden_states = hidden_states.view(-1, hidden_states.shape[-1])
    hidden_states = self.moe(hidden_states, topk_indices, topk_weights).view(*orig_shape)
    hidden_states = hidden_states + self.shared_experts(residuals)
    return hidden_states
```

