---
title: 为什么MLA需要解耦一部分RoPE
author: 66RING
date: 2024-10-7
tags: 
- machine learning
mathjax: true
---

# 为什么MLA需要解耦一部分RoPE

> 未拆分出head维度时, 从dim的视角看都是低秩投影
>
> MHA = [d] = [d] x [g*d_kv], g=1
>
> GQA = [d] = [d] x [g*d_kv], g=X, g*d_kv=d
>
> 而MLA是低秩投影后的工作

- abs
    * GQA本质也是一种低秩投影，只不过是简单的线性变化(拆分，复制)
    * MLA使用更贴合学习的方式做低秩投影，提升模型性能，再利用一些技巧兼容RoPE
    * 注意k的r部分用的还是x_i而不c_i


- GQA的后低秩处理:
    * group内合并成一个(分割，复制)
    * 只不过用的是简单的线性变换做低秩: 分割，复制
- MLA的后低秩处理 
    * MLA**使用一般的线性变化做低秩**, 增强模型的能力

## 矩阵吸收

- 隐空间表示kv: $c_i = x_i W_{c}$
    * 只需要缓存`c_i`
- 使用时解压: $k = c_i W_{uk}$
- 矩阵吸收: $q_i k_i = (x_t W_q) (c_i W_k)^T = x_t (W_q W_k^T) c_i^T$
    * aka 把$(W_q W_k^T)$取代原本q的投影矩阵
- cache更新: {ci} + xWc


## RoPE兼容: TODO

> RoPE会导致不能通过矩阵吸收合并成一个**固定的投影矩阵**, 矩阵吸收的效果就失效退化到GQA等情况

$(W_q W_k^T)$作为一个位置无关的矩阵当作q的投影矩阵，让引入rope时会出现一点问题。

- $q_i = x_i W_q R_i, k_j = c_j W_c R_j$
- 当qk相乘时就会自然的出现$R_{i-j}$这个表示相对位置的矩阵
    * $q_i k_j^T = (x_i W_q R_i) (c_j W_c R_j)^T = x_i (W_q R_i R_j^T W_c^T) c_i^T = x_i (W_q R_{i-j} W_c^T) c_i^T$
- 这时$(W_q R_{i-j} W_c^T)$不能合并成一个固定的投影矩阵，因为$R_{i-j}$是位置相关的

MLA的解决方法: Q, K增加dr个维度来添加RoPE信息。

$$
q_i = [ x_i W_{qc}, x_i W_{qr} R_i ] \in [dk + dr] \\
k_i = [ c_i W_{kc}, x_i W_{kr} R_i ] \in [dk + dr] \\
v_i = c_i W_v \in [dv] \\
s = qk \in [n, n] \\
o = sv \in [n, dv]
$$

注意k的r部分用的还是x_i而不c_i

## 最终优化

Q也低秩投影

$$
q_i = [ c'_i W_{qc}, c'_i W_{qr} R_i ] \in [dk + dr] \\
k_i = [ c_i W_{kc}, c_i W_{kr} R_i ] \in [dk + dr] \\
v_i = c_i W_v \in [dv] \\
s = qk \in [n, n] \\
o = sv \in [n, dv] \\
c'_i = x_i W_c' \\
c_i = x_i W_c
$$

注意k的r部分用的还是x_i而不c_i

推理阶段利用矩阵吸收的技巧, 把Wq和Wkc吸收到q的投影矩阵。此时计算规模为

$$
q_i \in [dc + dr] \\
k_i = [c_i, x_i W_kr R_i] \in [dc + dr]
$$


## 如何拆分multi head的

group内重复

multi head的通用表示`dk = headdim = [dg / h]`, 当g=h时为MHA, 当g=1时为MQA

dim一般会有dc = g(dk+dv) < d, 所以x到c是一个低秩投影。通过矩阵吸收, MLA计算过程中的headdim相当于dc, 比dk dv要大(文中说是4倍), 计算开销增加。但是kvcache少得多。roofline model。




