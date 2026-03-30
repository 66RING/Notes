---
title: 简单三步vllm
author: 66RING
date: 2025-03-12
tags: 
- vllm
- llm
mathjax: true
---

# 简单三步vllm

```python
def generate(model, input, max_new_tokens, kvcache):
    next_input = input
    generated_ids = []
    for i in range(max_new_tokens):
        # Stage 1: 构造输入
        outputs = model(next_input, kvcache)
        # Stage 2: 更新kvcache
        kvcache = outputs.kvcache
        # Stage 3: 更新输出
        next_input = logits_processor(outputs.logits)
        generated_ids.append(next_input)
```

- Stage 1: 构造输入
    * FlashAttentionMetadataBuilder
        + 构造slot mapping等metadata
    * flash attention + page table的原理
- Stage 2: 更新kvcache: 这时候可以看`cache_engine.py`和`flash_attn.py`了
    * 数据格式`|prefill_tokens | decoding_tokens|`, 然后从中间一截
    * block table: aka一个哈希表, seq id到list[block]的映射, 只不过使用tensor的形式连续存放
        + block_tables.shape = [max_batch_size, max_context_len // block_size] 
    * slot mapping
        + slot map是token在虚拟内存中的位置, 用于拷贝到kv_cache: 
            + cache更新相当于这样`key_cache.view(num_blocks * block_size, num_heads, head_dim)[slot_mapping] = key_states`
        + block table记录的是每个block的位置, 可以通过slot id // block size得到block id
            + 使用kvcache推理时直接用大块的key_cache + 索引找到对应的kvcache
    * cache engine
        + 只看cache的申请和形状即可
        + swap in, swap out暂时不用管
- Stage 3: 更新输出: 这时可以看Sequence类了
    * _process_model_outputs
        + STATE和计数更新: update_num_computed_tokens
            + prefill, decoding state切换
        + 记录新token id
            + process_outputs
    * 一个block用满后追加
        + _schedule() -> _schedule_running -> _append_slots


- 其他
    * block manager
        + 申请大块显存做table(就是一个tensor)
            + `(num_blocks, self.block_size, self.num_kv_heads, self.head_size)`
                + block_size就是一个block的大小，aka 能装几个token
        + 其他的block_manager都是一个逻辑id的管理, aka table索引
    * scheduler
        + 可以简单看成FIFO, 但是为了保证用户不等太久, 会限制每个seq最多一次处理的token数(包括prefill阶段)
            + `max_num_batched_tokens`管最多同时处理多少个token
            + `max_num_seqs`管最多同时处理多少个请求

