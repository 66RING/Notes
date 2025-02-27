---
title: RoPE CheatSheet
author: 66RING
date: 2024-02-27
tags: 
- marchine learning
mathjax: true
---

# rope

位置编码

对序列进行重新编码, $E = \{f(x_i, i)\}^N_{i=1}$。f是一个位置编码函数，接收单个token x和其位置i。

- tips
    * 是对一个向量(hidden state, (1, dim))进行编码，所以dim维度的信息也需要考虑

## 绝对位置编码

> 奇数dim用cos, 偶数dim用sin, 分子和token位置i有关，分母和dim有关

$x_i' = x_i + p_i$

其中

- $p_{i,2t} = sin(\frac{i}{10000^{\frac{2t}{d}}})$
- $p_{i,2t+1} = cos(\frac{i}{10000^{\frac{2t}{d}}})$

简单来说，要编码一个token(1, dim), 位置信息需要考虑两个维度的信息，一个是token当前的位置i, 另一个的dim维度元素的位置t, d的dim的大小。

## 旋转位置编码

> 对于词向量中的每个维度(1, dim)，两两一组操作

$x_i' = x_i e^{im\theta}$

```
|cos(mθ) -sin(mθ)| |q0|
|sin(mθ)  cos(mθ)| |q1|

最后简化成 =>

cos(mθ)q0  + -sin(mθ)q0
cos(mθ)q1  + sin(mθ)q1

=> aka
|cos(mθ)q0 | + |-sin(mθ)q0|
|cos(mθ)q1 | + | sin(mθ)q1|
...
|cos(mθ_i)q2i   | + |-sin(mθ_i)q2i  |
|cos(mθ_i)q2i+1 | + | sin(mθ_i)q2i+1|
```

即，cos组全正，sin组一半正一半负。**因为dim其实并没有先后顺序**，所以可以之前取前一般的dim正，后一半的dim负([:dim/2], [dim/2:])。

- m表示: 第m个字
- theta表示: 旋转角度，和dim相关的旋转角度
    * $\theta = \frac{1}{10000^{\frac{2i}{d}}}$, aka `theta = 1. / (base ** torch.range(0, dim, 2) / dim)`

对角线铺开就能实现高效的计算了，两两一组的

### code

- 计算theta
    * theta = base ^ (-2 i/d)
- 计算m theta
    * idx_theta = 

```python
q_embed = (q * cos) + (rotate_half(q) * sin)
k_embed = (k * cos) + (rotate_half(k) * sin)


# 其中
def rotate_half(x):
    """Rotates half the hidden dims of the input."""
    x1 = x[..., : x.shape[-1] // 2]
    x2 = x[..., x.shape[-1] // 2 :]
    return torch.cat((-x2, x1), dim=-1)

theta = 1. / (base ** torch.range(0, dim, 2) / dim) # aka inv_freq
# 批量算出每个位置的m_theta
m_theta = torch.einsum("i,j->ij", torch.range(max_seq_len), theta) # ada freq
emb = torch.cat((m_theta, m_theta), dim=-1)
cos = emb.cos()
sin = emb.sin()
```

### 形象化理解

对每个(dim)生成一个频率theta("波")来编码不同的dim。对于不同位置的token, dim的编码方式需要不同，所以theta乘上m修正一下。




