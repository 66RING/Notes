---
title: Mixtral MoE源码笔记
author: 66RING
date: 2024-05-10
tags: 
- machine leanring
mathjax: true
---

# Mixtral MoE源码笔记

> transformers/src/transformers/models/mixtral/modeling_mixtral.py
>
> 注意是mixtral不是mistral

和llama基本相同, 主要区别只在与MLP: 混合专家中的MLP有`num_experts`个mlp, 而llama只有一个mlp。核心代码在于`MixtralSparseMoeBlock`。

```python
class MixtralDecoderLayer(nn.Module):
    def __init__(self, config: MixtralConfig, layer_idx: int):
        super().__init__()
        self.hidden_size = config.hidden_size

        self.self_attn = MIXTRAL_ATTENTION_CLASSES[config._attn_implementation](config, layer_idx)

        self.block_sparse_moe = MixtralSparseMoeBlock(config)
        self.input_layernorm = MixtralRMSNorm(config.hidden_size, eps=config.rms_norm_eps)
        self.post_attention_layernorm = MixtralRMSNorm(config.hidden_size, eps=config.rms_norm_eps)

    def forward(
        self,
        hidden_states: torch.Tensor,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.LongTensor] = None,
        past_key_value: Optional[Tuple[torch.Tensor]] = None,
        output_attentions: Optional[bool] = False,
        output_router_logits: Optional[bool] = False,
        use_cache: Optional[bool] = False,
        **kwargs,
    ) -> Tuple[torch.FloatTensor, Optional[Tuple[torch.FloatTensor, torch.FloatTensor]]]:
        # ...

        # Self Attention
        hidden_states, self_attn_weights, present_key_value = self.self_attn(
            hidden_states=hidden_states,
            attention_mask=attention_mask,
            position_ids=position_ids,
            past_key_value=past_key_value,
            output_attentions=output_attentions,
            use_cache=use_cache,
        )
        hidden_states = residual + hidden_states

        # Fully Connected
        residual = hidden_states
        hidden_states = self.post_attention_layernorm(hidden_states)
        hidden_states, router_logits = self.block_sparse_moe(hidden_states)
        hidden_states = residual + hidden_states
        # ...
```


```python
class LlamaDecoderLayer(nn.Module):
    def __init__(self, config: LlamaConfig, layer_idx: int):
        super().__init__()
        self.hidden_size = config.hidden_size

        self.self_attn = LLAMA_ATTENTION_CLASSES[config._attn_implementation](config=config, layer_idx=layer_idx)

        self.mlp = LlamaMLP(config)
        self.input_layernorm = LlamaRMSNorm(config.hidden_size, eps=config.rms_norm_eps)
        self.post_attention_layernorm = LlamaRMSNorm(config.hidden_size, eps=config.rms_norm_eps)

    def forward(
        self,
        hidden_states: torch.Tensor,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.LongTensor] = None,
        past_key_value: Optional[Tuple[torch.Tensor]] = None,
        output_attentions: Optional[bool] = False,
        use_cache: Optional[bool] = False,
        cache_position: Optional[torch.LongTensor] = None,
        **kwargs,
    ) -> Tuple[torch.FloatTensor, Optional[Tuple[torch.FloatTensor, torch.FloatTensor]]]:

        # ...
        # Self Attention
        hidden_states, self_attn_weights, present_key_value = self.self_attn(
            hidden_states=hidden_states,
            attention_mask=attention_mask,
            position_ids=position_ids,
            past_key_value=past_key_value,
            output_attentions=output_attentions,
            use_cache=use_cache,
            cache_position=cache_position,
            **kwargs,
        )
        hidden_states = residual + hidden_states

        # Fully Connected
        residual = hidden_states
        hidden_states = self.post_attention_layernorm(hidden_states)
        hidden_states = self.mlp(hidden_states)
        hidden_states = residual + hidden_states
```

## MixtralSparseMoeBlock

MixtralSparseMoeBlock根据attention计算的结果`hidden_state`去选取topk个专家(mlp)

```python
# self.gate = nn.Linear(self.hidden_dim, self.num_experts, bias=False)
router_logits = self.gate(hidden_states)
routing_weights = F.softmax(router_logits, dim=1, dtype=torch.float)
routing_weights, selected_experts = torch.topk(routing_weights, self.top_k, dim=-1)

# NOTE: (batch * sequence_length, topk_experts, num_experts_one_hot)
# permute(2, 1, 0)
# => (num_experts_one_hot, topk_experts, batch * sequence_length)
expert_mask = torch.nn.functional.one_hot(selected_experts, num_classes=self.num_experts).permute(2, 1, 0)

# Loop over all available experts in the model and perform the computation on each expert
for expert_idx in range(self.num_experts):
    expert_layer = self.experts[expert_idx]
    # NOTE:取出当前专家负责的seqlen: top_x, 和专家id: idx
    idx, top_x = torch.where(expert_mask[expert_idx])

    if top_x.shape[0] == 0:
        continue

    # in torch it is faster to index using lists than torch tensors
    top_x_list = top_x.tolist()
    idx_list = idx.tolist()

    # Index the correct hidden states and compute the expert hidden state for
    # the current expert. We need to make sure to multiply the output hidden
    # states by `routing_weights` on the corresponding tokens (top-1 and top-2)
    # NOTE: hidden_states: (batch * sequence_length, hidden_dim)
    # NOTE: hidden_states[None, top_x_list] -> (1, top_x_list, hidden_dim)
    current_state = hidden_states[None, top_x_list].reshape(-1, hidden_dim)
    # NOTE: routing_weights.shape = (batch * sequence_length, n_experts)
    # 经过expert mlp后和专家权重做加权
    current_hidden_states = expert_layer(current_state) * routing_weights[top_x_list, idx_list, None]

    # However `index_add_` only support torch tensors for indexing so we'll use
    # the `top_x` tensor here.
    final_hidden_states.index_add_(0, top_x, current_hidden_states.to(hidden_states.dtype))
final_hidden_states = final_hidden_states.reshape(batch_size, sequence_length, hidden_dim)
return final_hidden_states, router_logits

```

因为`batch * seqlen`可能很大, 所以处于计算效率的考虑, 对`selected_expert`做`permute(2, 1, 0)`使得其形状变为: `(num_experts_one_hot, topk_experts, batch * sequence_length)`

之后, 处理流程为:

1. 遍历每个专家
2. 选出当前专家负责的`bs * seqlen`的索引信息: `idx, top_x = torch.where(expert_mask[expert_idx])`
    - `idx`为当前专家的index
    - `top_x`为当前专家负责的`bs * seqlen`
3. 选出当前专家负责的数据(`bs * seqlen`中选取)进行处理, 并根据选出的专家的权重进行加权
    - 选出数据`current_state = hidden_states[None, top_x_list].reshape(-1, hidden_dim)`
        * `hidden_states[None, top_x_list] -> (1, top_x_list, hidden_dim) -> current_state[top_x_list, hidden_dim]`
    - 专家处理`expert_layer(current_state)`
    - topk专家加权: `current_hidden_states = expert_layer(current_state) * routing_weights[top_x_list, idx_list, None]`
        * `routing_weights.shape = (batch * sequence_length, n_experts)`
4. 不断累加中间结果直到遍历完所有专家
    - `final_hidden_states.index_add_(0, top_x, current_hidden_states.to(hidden_states.dtype))`
        * `final_hidden_states.shape = (bs * seqlen, hidden_dim)`
        * 把当前专家计算得到的`bs * seqlen`维度的数据累加到最终结果的`bs * seqlen`维度的对应位置

核心:

- 多个mlp层加权
- 计算效率: 在`bs * seqlen`维度上并行
    * 遍历所有专家
    * 每个专家处理其负责的`bs * seqlen`维度上的数据




