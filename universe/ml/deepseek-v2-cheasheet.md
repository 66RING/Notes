---
title: DeepSeek-V2架构
author: 66RING
date: 2024-05-08
tags: 
- marchine learning
mathjax: true
---

# DeepSeek-V2架构

![architecture](https://github.com/deepseek-ai/DeepSeek-V2/blob/main/figures/architecture.png?raw=true)

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

- **QA**
    * 为什么会省kv cache
        + 类似MQA, 但kv进一步联合压缩压缩, `compressed_kv = self.kv_a_proj_with_mqa(hidden_states)`, Figure 3
            + low-rank kv 联合压缩
        + huggingface的`modeling_deepseek.py`中的kvcache保存仍然是保存完整的没有压缩过的cache
        + 所以在推理过程中, 如果保存全量cache则cache占用不变。如果保存压缩的cache则额外up proj的开销
        + Answer **就是用计算换空间, 或者说是计算和加载的tradeoff**, 因为随着硬件性能增强mem bound的影响更大, 所以保存压缩过后的cache虽然还需要一点up proj但是比没压过的up proj算得要快, 显存占用(需要加载的数据)也更小, 从而实现平衡


## 精髓

> 计算和加载的平衡: 降低了加载代价, 稍微提升了计算代价。make sense因为目前(2024年)的带宽速度提升比FLOP提升要慢。

利用联合低秩proj来降低显存, 降低加载代价, 而只稍微提升计算代价

- 如果全部不cache, 那可能会是compute bound, 瓶颈在硬件计算单元
- 如果全部做cache, 那可能会是memory bound, 瓶颈在从HBM加载

上面两个极端都不好。所以MLA保存的是压缩过后的cache(虽然开源版代码保存的是全量cache)，压缩过的cache(低秩联合proj了一部分)需要更少的显存(更少的加载)，而解压所需的proj开销也比从0开始proj要低。

这样，需要的compute稍微提升了，但需要的load更少了，从而实现平衡。tradeoff而不是走极端。

