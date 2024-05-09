---
title: DeepSeek-V2架构
author: 66RING
date: 2024-05-08
tags: 
- marchine learning
mathjax: true
---

# DeepSeek-V2架构

![architecture](https://github.com/deepseek-ai/DeepSeek-V2/blob/main/figures/architecture.png)

简单的说MLA + MoE

1. 参数嵌入更快: 利用类似lora的技术
```python
self.q_a_proj = nn.Linear(
    self.hidden_size, config.q_lora_rank, bias=config.attention_bias
)
self.q_a_layernorm = DeepseekV2RMSNorm(config.q_lora_rank)
self.q_b_proj = nn.Linear(
    config.q_lora_rank, self.num_heads * self.q_head_dim, bias=False
)

# NOTE: down -> norm -> up
q = self.q_b_proj(self.q_a_layernorm(self.q_a_proj(hidden_states)))
q = q.view(bsz, q_len, self.num_heads, self.q_head_dim).transpose(1, 2)
```
2. dim维度不是所有都需要位置编码: 一段用位置编码一段不用位置编码
```python
q_nope, q_pe = torch.split(
    q, [self.qk_nope_head_dim, self.qk_rope_head_dim], dim=-1
)
# NOTE: 只位置编码了一部分
q_pe, k_pe = apply_rotary_pos_emb(q_pe, k_pe, cos, sin, position_ids)
query_states[:, :, :, : self.qk_nope_head_dim] = q_nope
query_states[:, :, :, self.qk_nope_head_dim :] = q_pe
```
3. kv构造:
    - kv共用一个linear, 但输出维度足够k, v使用
    - value不pe, 和原始一样
    - key一部分pe, 另一部不pe
```python
compressed_kv = self.kv_a_proj_with_mqa(hidden_states)
# NOTE: 拆分出需要pe的那部分key
compressed_kv, k_pe = torch.split(
    compressed_kv, [self.kv_lora_rank, self.qk_rope_head_dim], dim=-1
)
k_pe = k_pe.view(bsz, q_len, 1, self.qk_rope_head_dim).transpose(1, 2)
# NOTE: [kv_lora_rank] -> k_nope + v
kv = (
    self.kv_b_proj(self.kv_a_layernorm(compressed_kv))
    .view(bsz, q_len, self.num_heads, self.qk_nope_head_dim + self.v_head_dim)
    .transpose(1, 2)
)
# NOTE: 拆分出k, v
k_nope, value_states = torch.split(
    kv, [self.qk_nope_head_dim, self.v_head_dim], dim=-1
)
q_pe, k_pe = apply_rotary_pos_emb(q_pe, k_pe, cos, sin, position_ids)
# NOTE: 一部分用(pe)position embedding, 一部分不用pe
key_states[:, :, :, : self.qk_nope_head_dim] = k_nope
key_states[:, :, :, self.qk_nope_head_dim :] = k_pe
```

## MLA: Multi-Head Latent Attention

attention算子部分不变, 只不过`query_state`, `key_state`如上述的构造方法: 一部分pe, 一部分不pe。




