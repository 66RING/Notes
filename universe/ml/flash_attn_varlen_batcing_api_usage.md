---
title: Flash attention变长batching API使用
author: 66RING
date: 2024-05-31
tags: 
- machine learning
- llm
mathjax: true
---

# Flash attention变长batching API使用

主要记录`flash_attn.flash_attn_varlen_func`这个接口的使用, 精髓在于理解**函数签名**和**输入形状**: 函数签名需要每个seq的offset, 输入形状需要`(bs, seqlen)`平坦化后的`(total_num, nhead, headdim)`

```python
from flash_attn import flash_attn_varlen_func
```

需要注意使用`causal`这个参数才能进入causal模式哦。

```python
def flash_attn_varlen_func(
    q, # (total_q, nheads, headdim)
    k, # (total_k, nheads, headdim)
    v, # (total_v, nheads, headdim)
    cu_seqlens_q, # (batch_size + 1)
    cu_seqlens_k, # (batch_size + 1)
    max_seqlen_q, # 所有序列中最长的q的长度
    max_seqlen_k, # 所有序列中最长的k的长度
    dropout_p=0.0,
    softmax_scale=None,
    causal=False,
    window_size=(-1, -1),  # -1 means infinite context window
    return_attn_probs=False,
):
    pass
```

值得注意的是qkv输入形状上需要是`(total_num, nheads, headdim)`而不是`(batch_size, seqlen, nheads, headdim)`, 和`flash_attn_func`是不同的。这是因为在变长batching中把`bs * seqlen`打平展开了，然后再结合offset去找到每个batch的其实位置做计算。

最重要的参数就是`cu_seqlens_q`和`cu_seqlens_k`, 用于记录找到每个batch需要的offset。比如seq0的offset=0, seq1的offset=seq0.len, seq2的offset=seq0.len+seq1.len, 因此就**是一个不包含自身的前缀和**, 可以通过`torch.cumsum`减去各自的seqlen获得:

```python
prefill_start_pos = torch.cumsum(seq_len, dim=0, dtype=torch.int32) - seq_len
```

有因为API要求的`cu_seqlens_k`的形状的`batch_size+1`还需在末尾追加一个"总token数":

```python
prefill_start_pos = torch.cat([prefill_start_pos, torch.tensor([torch.sum(seq_len)], dtype=torch.int32, device="cuda")], dim=0)
```

完整示例如下:

## Demo

```python
import torch
from flash_attn import flash_attn_varlen_func, flash_attn_func

def main():
    dtype = torch.bfloat16
    HEAD = 2
    HEAD_DIM = 2
    seqlens = [1, 2, 3, 4]
    query = torch.empty(0, HEAD, HEAD_DIM, dtype=dtype).cuda()
    key = torch.empty(0, HEAD, HEAD_DIM, dtype=dtype).cuda()
    value = torch.empty(0, HEAD, HEAD_DIM, dtype=dtype).cuda()

    querys = []
    keys = []
    values = []
    for l in seqlens:
        q = torch.rand(l, HEAD, HEAD_DIM, dtype=dtype).cuda()
        k = torch.rand(l, HEAD, HEAD_DIM, dtype=dtype).cuda()
        v = torch.rand(l, HEAD, HEAD_DIM, dtype=dtype).cuda()
        querys.append(q)
        keys.append(k)
        values.append(v)
        query = torch.cat([query, q], dim=0)
        key = torch.cat([key, k], dim=0)
        value = torch.cat([value, v], dim=0)

    print("===Standard===")
    for q, k, v in zip(querys, keys, values):
        q = q.unsqueeze(0)
        k = k.unsqueeze(0)
        v = v.unsqueeze(0)
        out = flash_attn_func(q, k, v)
        print(out)
    print("=========\n")

    seq_len = torch.tensor(seqlens, dtype=torch.int32).cuda()
    # NOTE: flash_attn_varlen_func这个接口需要(bs + 1)长度的cu_seqlens_q和cu_seqlens_k
    prefill_start_pos = torch.cumsum(seq_len, dim=0, dtype=torch.int32) - seq_len
    prefill_start_pos = torch.cat([prefill_start_pos, torch.tensor([torch.sum(seq_len)], dtype=torch.int32, device="cuda")], dim=0)
    print(prefill_start_pos.shape)
    print(prefill_start_pos)

    print(query.shape, key.shape, value.shape)
    cu_seqlens_q = prefill_start_pos
    cu_seqlens_k = prefill_start_pos
    max_seqlen_q = max(seqlens)
    max_seqlen_k = max(seqlens)

    out = flash_attn_varlen_func(query, key, value, cu_seqlens_q, cu_seqlens_k, max_seqlen_q, max_seqlen_k)
    acc = 0

    print("===Varlen===")
    for l in seqlens:
        print(out[acc:acc+l])
        acc += l
    print("=========\n")

if __name__ == "__main__":
    main()
```

