---
title: DeepSeek-V2架构
author: 66RING
date: 2024-05-08
tags: 
- machine learning
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
# compressed_kv取了low_rank(hidden_size)中的kv_lora_rank个维度. shape=(..., :kv_lora_rank)
# k_pe取了low_rank(hidden_size)中的qk_rope_head_dim个维度. shape=(..., -qk_rope_head_dim:)
# kv_lora_rank+qk_rope_head_dim=hidden_size
# NOTE: kvcache只需要缓存compressed_kv和k_pe
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

兼容了mla的huggingface modeling实现: https://huggingface.co/deepseek-ai/DeepSeek-V2-Chat/discussions/12/files

> 本质: 对hidden_size维度做了low_rank压缩(hidden_size包含multi head的维度, 所以可以和GQA等等价上), 然后只需要缓存low rank的cache。aka除了GQA压缩head维度外, 还压缩了hidden_size维度。最后通过重算还原出原始的hidden_size

attention算子部分不变, 只不过`query_state`, `key_state`如上述的构造方法: 一部分pe, 一部分不pe。

- **QA**
    * 为什么会省kv cache
        + 类似MQA, 但kv进一步联合压缩压缩, `compressed_kv = self.kv_a_proj_with_mqa(hidden_states)`, Figure 3
            + low-rank kv 联合压缩
        + huggingface的`modeling_deepseek.py`中的kvcache保存仍然是保存完整的没有压缩过的cache
        + 所以在推理过程中, 如果保存全量cache则cache占用不变。如果保存压缩的cache则额外up proj的开销
        + Answer **就是用计算换空间, 或者说是计算和加载的tradeoff**, 因为随着硬件性能增强mem bound的影响更大, 所以保存压缩过后的cache虽然还需要一点up proj但是比没压过的up proj算得要快, 显存占用(需要加载的数据)也更小, 从而实现平衡
            - **只cache联合压缩后的结果，需要时再重算出来**
- 另外，高性能算子的实现技巧/优化在于矩阵吸收(结合率): https://zhuanlan.zhihu.com/p/700214123
    - 简单的说可以是: (q * Wq) * (Wc * kv) 可以规约成(q * Wq * Wc) * kv, 从而减少计算量
    - q, k中的吸收: $(c_q \cdot W_{UQ}) \cdot (W_{UK} \cdot c_{kv}) = (c_q \cdot W_{UQ} \cdot W_{UK}) \cdot c_{kv}$
    - v, o中的吸收: $(p \cdot (c_{kv} \cdot W_{UV})) \cdot W_o = (p \cdot c_{kv}) \cdot (W_{UV} \cdot W_o)$


### code

> https://huggingface.co/deepseek-ai/DeepSeek-V2-Chat/discussions/12/files

```python
class DeepseekV2Attention(nn.Module):
    """Multi-headed attention from 'Attention Is All You Need' paper"""

    def __init__(self, config: DeepseekV2Config, layer_idx: Optional[int] = None):
        super().__init__()
        self.config = config
        self.layer_idx = layer_idx
        if layer_idx is None:
            logger.warning_once(
                f"Instantiating {self.__class__.__name__} without passing `layer_idx` is not recommended and will "
                "to errors during the forward call, if caching is used. Please make sure to provide a `layer_idx` "
                "when creating this class."
            )

        self.attention_dropout = config.attention_dropout
        self.hidden_size = config.hidden_size
        self.num_heads = config.num_attention_heads

        self.max_position_embeddings = config.max_position_embeddings
        self.rope_theta = config.rope_theta
        self.q_lora_rank = config.q_lora_rank
        self.qk_rope_head_dim = config.qk_rope_head_dim
        self.kv_lora_rank = config.kv_lora_rank
        self.v_head_dim = config.v_head_dim
        self.qk_nope_head_dim = config.qk_nope_head_dim
        self.q_head_dim = config.qk_nope_head_dim + config.qk_rope_head_dim

        self.is_causal = True

        self.q_a_proj = nn.Linear(
            self.hidden_size, config.q_lora_rank, bias=config.attention_bias
        )
        self.q_a_layernorm = DeepseekV2RMSNorm(config.q_lora_rank)
        self.q_b_proj = nn.Linear(
            config.q_lora_rank, self.num_heads * self.q_head_dim, bias=False
        )

        self.kv_a_proj_with_mqa = nn.Linear(
            self.hidden_size,
            config.kv_lora_rank + config.qk_rope_head_dim,
            bias=config.attention_bias,
        )
        self.kv_a_layernorm = DeepseekV2RMSNorm(config.kv_lora_rank)
        self.kv_b_proj = nn.Linear(
            config.kv_lora_rank,
            self.num_heads
            * (self.q_head_dim - self.qk_rope_head_dim + self.v_head_dim),
            bias=False,
        )

        self.o_proj = nn.Linear(
            self.num_heads * self.v_head_dim,
            self.hidden_size,
            bias=config.attention_bias,
        )
        self._init_rope()

        self.softmax_scale = self.q_head_dim ** (-0.5)
        if self.config.rope_scaling is not None:
            mscale_all_dim = self.config.rope_scaling.get("mscale_all_dim", 0)
            scaling_factor = self.config.rope_scaling["factor"]
            if mscale_all_dim:
                mscale = yarn_get_mscale(scaling_factor, mscale_all_dim)
                self.softmax_scale = self.softmax_scale * mscale * mscale

    def _init_rope(self):
        if self.config.rope_scaling is None:
            self.rotary_emb = DeepseekV2RotaryEmbedding(
                self.qk_rope_head_dim,
                max_position_embeddings=self.max_position_embeddings,
                base=self.rope_theta,
            )
        else:
            scaling_type = self.config.rope_scaling["type"]
            scaling_factor = self.config.rope_scaling["factor"]
            if scaling_type == "linear":
                self.rotary_emb = DeepseekV2LinearScalingRotaryEmbedding(
                    self.qk_rope_head_dim,
                    max_position_embeddings=self.max_position_embeddings,
                    scaling_factor=scaling_factor,
                    base=self.rope_theta,
                )
            elif scaling_type == "dynamic":
                self.rotary_emb = DeepseekV2DynamicNTKScalingRotaryEmbedding(
                    self.qk_rope_head_dim,
                    max_position_embeddings=self.max_position_embeddings,
                    scaling_factor=scaling_factor,
                    base=self.rope_theta,
                )
            elif scaling_type == "yarn":
                kwargs = {
                    key: self.config.rope_scaling[key]
                    for key in [
                        "original_max_position_embeddings",
                        "beta_fast",
                        "beta_slow",
                        "mscale",
                        "mscale_all_dim",
                    ]
                    if key in self.config.rope_scaling
                }
                self.rotary_emb = DeepseekV2YarnRotaryEmbedding(
                    self.qk_rope_head_dim,
                    max_position_embeddings=self.max_position_embeddings,
                    scaling_factor=scaling_factor,
                    base=self.rope_theta,
                    **kwargs,
                )
            else:
                raise ValueError(f"Unknown RoPE scaling type {scaling_type}")

    def _shape(self, tensor: torch.Tensor, seq_len: int, bsz: int):
        return (
            tensor.view(bsz, seq_len, self.num_heads, self.v_head_dim)
            .transpose(1, 2)
            .contiguous()
        )

    def forward(
        self,
        hidden_states: torch.Tensor,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.LongTensor] = None,
        past_key_value: Optional[Cache] = None,
        output_attentions: bool = False,
        use_cache: bool = False,
        **kwargs,
    ) -> Tuple[torch.Tensor, Optional[torch.Tensor], Optional[Tuple[torch.Tensor]]]:
        if "padding_mask" in kwargs:
            warnings.warn(
                "Passing `padding_mask` is deprecated and will be removed in v4.37. Please make sure use `attention_mask` instead.`"
            )
        bsz, q_len, _ = hidden_states.size()

        # NOTE: q_b_proj: 解压
        # [..., lora] -> [..., dim]
        q = self.q_b_proj(self.q_a_layernorm(self.q_a_proj(hidden_states)))
        q = q.view(bsz, q_len, self.num_heads, self.q_head_dim).transpose(1, 2)
        # NOTE: [..., nope_dim] [..., pe_dim]
        q_nope, q_pe = torch.split(
            q, [self.qk_nope_head_dim, self.qk_rope_head_dim], dim=-1
        )

        # NOTE: 压缩: [..., dim] -> [..., lora + pe_dim]
        compressed_kv = self.kv_a_proj_with_mqa(hidden_states)
        # NOTE: [..., lora] [..., pe_dim]
        compressed_kv, k_pe = torch.split(
            compressed_kv, [self.kv_lora_rank, self.qk_rope_head_dim], dim=-1
        )
        compressed_kv = self.kv_a_layernorm(compressed_kv)
        k_pe = k_pe.view(bsz, q_len, 1, self.qk_rope_head_dim).transpose(1, 2)

        kv_seq_len = k_pe.shape[-2]
        if past_key_value is not None:
            if self.layer_idx is None:
                raise ValueError(
                    f"The cache structure has changed since version v4.36. If you are using {self.__class__.__name__} "
                    "for auto-regressive decoding with k/v caching, please make sure to initialize the attention class "
                    "with a layer index."
                )
            kv_seq_len += past_key_value.get_usable_length(kv_seq_len, self.layer_idx)

        cos, sin = self.rotary_emb(q_pe, seq_len=kv_seq_len)
        q_pe, k_pe = apply_rotary_pos_emb(q_pe, k_pe, cos, sin, position_ids)

        if past_key_value is not None:
            cache_kwargs = {"sin": sin, "cos": cos}  # Specific to RoPE models
            # NOTE: !!! 不解压compressed_kv.shape = [bsz, seqlen, lora_dim], 不是lora_headdim
            compressed_kv = compressed_kv.unsqueeze(1)
            k_pe, compressed_kv = past_key_value.update(k_pe, compressed_kv, self.layer_idx, cache_kwargs)
            compressed_kv = compressed_kv.squeeze(1)
        
        # NOTE: !!! 矩阵吸收
        # NOTE: kv_b_proj, 解压矩阵[..., lora] -> [h * (nope_dim + v_dim)]
        # (CqUq @ (CkUK).T) @ CvUv
        # 1. Cq(UqUk.T)Ck.T @ CvUv
        kv_b_proj = self.kv_b_proj.weight.view(self.num_heads, -1, self.kv_lora_rank)
        # NOTE: [nope_dim, lora], 注意是matmul乘转职
        q_absorb = kv_b_proj[:, :self.qk_nope_head_dim,:]
        out_absorb = kv_b_proj[:, self.qk_nope_head_dim:, :]

        q_nope = torch.matmul(q_nope, q_absorb)
        # NOTE: 
        # 1. pe attn -> 常规计算
        # 2. nope attn -> 矩阵吸收的fuse: [..., nope] @ [lora, nope] @ [kv_len, lora]
        # 3. o = score @ CvUv -> score @ Cv @ Uv
        #   [kvlen, lora] @ [nope, lora]
        attn_weights = (torch.matmul(q_pe, k_pe.mT) + torch.matmul(q_nope, compressed_kv.unsqueeze(-3).mT)) * self.softmax_scale
        if attn_weights.size() != (bsz, self.num_heads, q_len, kv_seq_len):
            raise ValueError(
                f"Attention weights should be of size {(bsz, self.num_heads, q_len, kv_seq_len)}, but is"
                f" {attn_weights.size()}"
            )
        assert attention_mask is not None
        if attention_mask is not None:
            if attention_mask.size() != (bsz, 1, q_len, kv_seq_len):
                raise ValueError(
                    f"Attention mask should be of size {(bsz, 1, q_len, kv_seq_len)}, but is {attention_mask.size()}"
                )
            attn_weights = attn_weights + attention_mask

        # upcast attention to fp32
        attn_weights = nn.functional.softmax(
            attn_weights, dim=-1, dtype=torch.float32
        ).to(q_pe.dtype)
        attn_weights = nn.functional.dropout(
            attn_weights, p=self.attention_dropout, training=self.training
        )
        # NOTE: [..., qlen, kvlen] @ [..., kv_len, lora] -> [..., qlen, lora]
        attn_output = torch.einsum('bhql,blc->bhqc', attn_weights, compressed_kv)

        attn_output = torch.matmul(attn_output, out_absorb.mT) 

        if attn_output.size() != (bsz, self.num_heads, q_len, self.v_head_dim):
            raise ValueError(
                f"`attn_output` should be of size {(bsz, self.num_heads, q_len, self.v_head_dim)}, but is"
                f" {attn_output.size()}"
            )

        attn_output = attn_output.transpose(1, 2).contiguous()

        attn_output = attn_output.reshape(bsz, q_len, self.num_heads * self.v_head_dim)

        attn_output = self.o_proj(attn_output)

        if not output_attentions:
            attn_weights = None

        return attn_output, attn_weights, past_key_value
```

## 精髓

> 计算和加载的平衡: 降低了加载代价, 稍微提升了计算代价。make sense因为目前(2024年)的带宽速度提升比FLOP提升要慢。

利用联合低秩proj来降低显存, 降低加载代价, 而只稍微提升计算代价

- 如果全部不cache, 那可能会是compute bound, 瓶颈在硬件计算单元
- 如果全部做cache, 那可能会是memory bound, 瓶颈在从HBM加载

上面两个极端都不好。所以MLA保存的是压缩过后的cache(虽然开源版代码保存的是全量cache)，压缩过的cache(低秩联合proj了一部分)需要更少的显存(更少的加载)，而解压所需的proj开销也比从0开始proj要低。

这样，需要的compute稍微提升了，但需要的load更少了，从而实现平衡。tradeoff而不是走极端。

