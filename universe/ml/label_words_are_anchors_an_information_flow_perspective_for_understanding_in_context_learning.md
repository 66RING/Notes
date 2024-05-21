---
title: Label Words are Anchors An Information Flow Perspective for Understanding In-Context Learning深度解析
author: 66RING
date: 2024-05-21
tags: 
- machine learning
mathjax: true
---

# 深入理解Label Words are Anchors: An Information Flow Perspective for Understanding In-Context Learning

## abs

> 只测试了GPT模型

- 探究了ICL(in context learning)如何学习上下文的机制
    * 提出"Information Flow with Labels as Anchors"假说
- 发现靠前的layer主要做aggregation来聚合信息, 靠后的layer不那么aggregation, 更多的提取功能
    * Figure 1, Figure 2
- 提出anchor re-weighting方法和压缩方法来加速推理
- 1.8x推理提速

## background：什么是ICL

我对ICL的理解: 提问前举例子。不用更新模型权重就能让模型学到东西。和few-shot的区别在于few-shot的举例是在finetune阶段的, 即是会更新模型权重的举例说明。

## 假设的证明

在实际应用上完全不需要复杂的score计算，这得益于ICL的机制。

假设:

- 浅层layer负责聚合
- 深层layer负责提取

## 显著性得分Saliency Scores

attn_map或者说attn score或者说attn weight与loss对attn的导(利用torch的自动求导机制)的累加

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/saliece_score.png)

代码实现

1. 为了能对attn求导，则让attn乘上一个占位参数，反向时参数就会自动求导并和attn相乘

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/score_impl.png)

2. 因为上图在自动求导(反向)完成后就会自动形成Sigma右边的结构，所以使用时只需要sum和abs即可

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/score_usage.png)

这是显著性的计算, 但在实际应用中是用不到显著性计算的，直接利用ICL中的lable id提取即可。

### 验证浅层layer的影响

#### 基于信息流的验证

- 在浅层block掉label位置的信息聚合流(attn weight)影响大, 在深层block掉信息聚合流(attn weight)影响小, 说明只有浅层有聚合作用, 深层几乎没有影响
- 随机block的影响不及对lable的精确block，说明确实lable的聚合作用比较重要

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/info_flow_block.png)

阻塞信息流的代码实现：修改attention weight(attn score)，把label对上文的注意力全部mask掉

```python
def _forward(self, attn_weights):
    class_poss, final_pos = self.get_pos(self.input_ids, self.label_id_dict, self.pad_token_id)
    bsz, sql = self.input_ids.shape
    for class_pos in class_poss:
        for b_idx in range(bsz):
            for single_class_pos in class_pos[b_idx]:
                # NOTE: (bs, head, seqlen, seqlen)
                attn_weights[b_idx, :,
                # NOTE: mask掉lable id一个范围(window_size)内的attention
                single_class_pos: single_class_pos + self.window_size + 1,
                :single_class_pos] = -10000.
    return attn_weights
```

#### 基于显著性得分的验证

代码实现

1. 根据上一节的方法知道显著性的计算方法后这节对比各个阶段显著性的主导地位class_pos表示lable在输入序列的位置

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_proportion_lab.png)

2. proportion1,2,3分别是S_wp(信息流向label), S_pq(label流向目标预测token), S_ww(其他信息流)

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_proportion_equation.png)

3. 可见浅层时S_wp占主导(信息流向label), 深层时S_pq占主导(目标预测从lable中提取), 其他情况(S_ww)影响不大

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_proportion_result.png)

### 验证深层layer的影响

思路: 模型的最终预测结果应该和lable有很强的相关性(a strong correlation between the attention distributions on the label words of the target position and the model’s final prediction)

所以引入了AUCROC得分来衡量模型输出和lable的attention的相关性。并且为了为了验证l层累积的效果还引入了Rl(eq 5), 并设置了一个0.5的threshold

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_aucroc_score_equation.png)

结果深层layer的相关性更高，达到0.8，而浅层layer相关性低

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_aucroc_result.png)

代码实现

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_aucroc_result.png)


## 假设的应用

Anchor-reweighting

> Each β0 i is a learnable parameter, set uniquely for different attention heads and layers.
Attn weight直接乘上一个可学习权重，给重要的东西多关注一点，见Appendix G

代码实现：

```python
class AttentionAdapter(AttentionAdapterBase):
    def _forward(self, attn_weights):
        # NOTE: 给重要的东西多关注一点, final_poss <- class_poss
        class_poss = self.class_poss
        final_poss = self.final_poss
        weight = self.weight.exp()
        bsz, n_head, seq_len, _ = attn_weights.shape
        assert bsz == 1
        mask_mat = torch.ones((1, n_head, seq_len, seq_len), device=attn_weights.device)
        mask_mat[:, :, final_poss, class_poss] = weight.reshape(1, self.n_head, self.n_demo)
        # NOTE: re-weighting 乘上一个可学习的参数
        return attn_weights * mask_mat
```

Compress

还是利用ICL的特性，示例 + 正文

用标签位置的hidden state来代表整个示例(ICL中的示例概念)

![](https://raw.githubusercontent.com/66RING/Notes/master/universe/ml/assets/img/anchor_hidden_compress.png)

- Text_anchor直接取文本的标签 => 不行
- Hidden_anchor取标签对应的hidden state => 行

代码实现：只保留lable所在位置的kvcache。但如果不是ICL没有lable怎么办

```python
# 根据ICL的模板(示例 + 正文)可以提取到lable        
mask = context_solver.get_mask(data[0]['input_ids'])
past_key_values = tuple(
            tuple(t[:, :, mask, :] for t in tup) for tup in past_key_values)
```


