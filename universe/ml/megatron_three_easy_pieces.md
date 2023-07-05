---
title: 简单三步看清Megatron-LM的实现, Megatron源码解析
author: 66RING
date: 2023-07-02
tags: 
- machine learning
- distributed
mathjax: true
---

# Megatron TEP

> 小白帮小白, 从我一个小白的视角记录我想要知道的东西, 希望能"模式匹配"帮助下一个小白
>
> Megatron源码解析(overview版)
>
> 所谓简单三步就是: 数据并行, 流水并行, 张量并行

这里将简单理清Megatron实现数据并行, 流水并行, 张量并行的整体逻辑, 但talk is cheap, 更详细的代码细节可以看完本文后分模块再去深究。

- 数据并行: 分布式文件/数据系统
- 流水并行: P2P通信
- 张量并行: 人工算子拆分

最后把三个模块串起来。


## 数据并行

**数据并行的本质的DDL**。DDL(Distributed Data Parallel - PyTorch)可以理解为一种"分布式数据系统", 用户访问数据无需关心数据位置, 可以简单`data[index]`访问任意节点中的数据。PyTorch提供了这么系统(Sampler), 并且通过分配让不同节点看到的数据不相交。如:

```
data: a b c d
=> 
node0: a c
node1: b d
```

实例代码如下: 在DistributedSampler让数据平均分配到GPU上

```python
# run: CUDA_VISIBLE_DEVICES=0,1 python -m torch.distributed.launch --nproc_per_node=2 test.py

import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
from torch.utils.data.distributed import DistributedSampler
torch.distributed.init_process_group(backend="nccl")
 
batch_size = 4
data_size = 8
 
local_rank = torch.distributed.get_rank()
torch.cuda.set_device(local_rank)
device = torch.device("cuda", local_rank)
 
class RandomDataset(Dataset):
        def __init__(self, length, local_rank):
            self.len = length
            self.data = torch.stack([torch.ones(1), torch.ones(1)*2,torch.ones(1)*3,torch.ones(1)*4,torch.ones(1)*5,torch.ones(1)*6,torch.ones(1)*7,torch.ones(1)*8]).to('cuda')
            self.local_rank = local_rank
        def __getitem__(self, index):
            return self.data[index]
        def __len__(self):
            return self.len
 
dataset = RandomDataset(data_size, local_rank)
sampler = DistributedSampler(dataset)
 
rand_loader = DataLoader(dataset=dataset,batch_size=batch_size,sampler=sampler)
epoch = 0
while epoch < 2:
    sampler.set_epoch(epoch)
    if epoch == 0:
        for data in rand_loader:
                print(data)
    epoch+=1

'''
输出: GPU间的数据不重叠
tensor([[5.],
        [8.],
        [3.],
        [2.]], device='cuda:0')
tensor([[1.],
        [4.],
        [6.],
        [7.]], device='cuda:1')
'''
```



## 流水并行

**流水并行的本质就是状态机, 对应到Megatron中是P2P通信**。抛开分布式训练不谈, 单纯的流水线执行可以怎么实现? 是不是可以一个状态机, 前驱节点完成后数据传输到下一个节点的输入, 然后通知下一个节点就绪。Megatron中就是这么实现的: 首先是状态机标注谁该收谁该发, 然后使用P2P通信作为传输和就绪的信号。

对应代码如下, 调用流程为: `train() -> train_step() forward_backward_func() -> forward_backward_pipelining_with_interleaving()/forward_backward_pipelining_without_interleaving() -> send_forward_recv_forward() -> _communicate() -> _p2p_ops()`, 以`_p2p_ops`为例

```python
def _p2p_ops(*,
             tensor_send_prev: Optional[torch.Tensor],
             tensor_recv_prev: Optional[torch.Tensor],
             tensor_send_next: Optional[torch.Tensor],
             tensor_recv_next: Optional[torch.Tensor],
             group: torch.distributed.ProcessGroup):
    reqs = []
    rank = get_pipeline_model_parallel_rank()
    # 状态机
    if get_pipeline_model_parallel_rank() % 2 == 0:
        if tensor_send_next is not None:
            # P2P发送
            send_next_req = torch.distributed.isend(
                tensor=tensor_send_next,
                dst=get_pipeline_model_parallel_next_rank(),
                group=group,
            )
            reqs.append(send_next_req)

        # 状态机
        if tensor_recv_prev is not None:
            # ......

    else:
        if tensor_recv_prev is not None:
            recv_prev_req = torch.distributed.irecv(
                tensor=tensor_recv_prev,
                src=get_pipeline_model_parallel_prev_rank(),
                group=group,
            )
            reqs.append(recv_prev_req)

        # ....
    return reqs

# 相当于p2p_func中构建了状态机的执行计划, 然后会在_communicate中wait()等待通知并做下一步
def _communicate():
    # ...
    # Send tensors in both the forward and backward directions as appropriate.
    if config.use_ring_exchange_p2p:
        def _ring_exchange_wrapper(**kwargs):
            torch.distributed.ring_exchange(**kwargs)
            return []
        p2p_func = _ring_exchange_wrapper
    elif config.batch_p2p_comm:
        assert wait_on_reqs
        p2p_func = _batched_p2p_ops
    else:
        p2p_func = _p2p_ops

    reqs = p2p_func(tensor_send_prev=tensor_send_prev,
                    tensor_recv_prev=tensor_recv_prev,
                    tensor_send_next=tensor_send_next,
                    tensor_recv_next=tensor_recv_next,
                    group=get_pipeline_model_parallel_group())

    if wait_on_reqs and len(reqs) > 0:
        for req in reqs:
            req.wait()
        reqs = None

```


## 张量并行

Megatron中张量并行的本质就是手动的拆分算子。原理如下:

```
a11, a12       b11, b12        a11*b11 + a12*b21,  a11*b12 + a12*b22
           x              =   
a21, a22       b21, b22        a21*b11 + a22*b21,  a21*b12 + a22*b22
```

如果从`+`分成两份会发现

```
a11, ___       b11, b12        a11*b11 + 0,  a11*b12 + 0
           x              =   
a21, ___       ___, ___        a21*b11 + 0,  a21*b12 + 0


___, a12       ___, ___        0 + a12*b21,  0 + a12*b22
           x              =   
___, a22       b21, b22        0 + a22*b21,  0 + a22*b22
```

刚好能刚好能切分出`+`两边的值, 最后在经过一个 $\Sigma$ 就得到原来的结果。基于这个原理多维的行列式就可以拆分成多组进行独立运行, 如分配到不同GPU中运算, 最后通过一个Reduce操作(即$\Sigma$ 加和)就在一个GPU上能得到最终值。为了让所有GPU都得到最终值, 那就是让其他GPU也"执行reduce", 这个过程就叫作all reduce。而每个GPU依次执行reduce, 就是all reduce的一种all to all的实现(比较著名的还有ring, tree的实现)。

抽象一下上述流程可以是: tensor(input) =split=> slice => compute => (all)reduce => result。

一个self attention qkv计算的例子

```python
class ParallelAttention(MegatronModule):
    def __init__(...):
        # ...
        if attention_type == AttnType.self_attn:
            self.query_key_value = tensor_parallel.ColumnParallelLinear(
                config.hidden_size,
                3 * projection_size,
                config=config,
                init_method=config.init_method,
                bias=args.add_bias_linear,
                gather_output=False)

    def forward():
        # ...
        # =====================
        # Query, Key, and Value
        # =====================

        if self.attention_type == AttnType.self_attn:
            # Attention heads [sq, b, h] --> [sq, b, (np * 3 * hn)]
            mixed_x_layer, _ = self.query_key_value(hidden_states)
```

之后`query_key_value`就会调用人工手动写好的"拆分整合一条龙算子": `query_key_value -> ColumnParallelLinear:forward -> linear_with_grad_accumulation_and_async_allreduce`

其他的"人工算子"还有`RowParallelLinear`, `VocabParallelEmbedding`等。主要会集中在`core/tensor_parallel/`


## 串讲


数据并行在最底层将数据均匀分配到不同GPU上。每份数据将映射到一组GPU上做流水并行。每个流水线上的任务内部使用人工的算子拆分将tensor的处理交给节点内(intra)的多个GPU并行处理再合并。

假设数据并行dp=2, 流水并行pp=2, 张量并行tp=2。则理想的总的显卡需求为2x2x2=8

```
数据并行
dp0: g0 g1 g2 g3
dp1: g4 g5 g6 g7

单份数据的流水并行
dp0.pp0: [g0 g1] =pipe=> [g2 g3]
dp0.pp1: [g4 g5] =pipe=> [g6 g7]

单个流水线任务的张量并行
pp0.tp0: g0
pp0.tp1: g1

最后逐层聚合出完整的数据
reduce(pp0.tp0, pp0.tp1) => [g0 g1]
batch(dp0.pp0, dp0. pp1) => [g0 g1 g2 g3]
gather(dp0, dp1) => [g0 g1 g2 g4 g4 g5 g6 g7]
```


### (optional)具体分组情况

- 分组: 本质就是range切分, n张卡, (p, t, d)表示流水线, tensor, 数据并行度
    * data p groups
        + 先n / p分大组(和pipeline group对应), 然后组内等间隔t取值分data p小组
    * tensor p groups
        + 均分成一个个区间即可
        + rank一定相邻
    * pipeline p group
        + 也是均分成一个个区间
    * 总体的, 从overview看:
        + 先是数据并行分组, 多个显卡处理一个数据切分, 看成整体的一块显卡
        + 数据切分组内, 在用流水线并行的方式处理这个整体是任务
        + **至于tensor并行就比较微妙**, 它的rank和数据并行分组应该满足某种倍数关系, 否则就浪费??笔者不太确定, **NOTE: 非常微妙**

```python
data_parallel_size: int = world_size // (tensor_model_parallel_size *
                                         pipeline_model_parallel_size)

num_tensor_model_parallel_groups: int  = world_size // tensor_model_parallel_size
num_pipeline_model_parallel_groups: int = world_size // pipeline_model_parallel_size

# Build the data-parallel groups.
global _DATA_PARALLEL_GROUP
global _DATA_PARALLEL_GLOBAL_RANKS
assert _DATA_PARALLEL_GROUP is None, 'data parallel group is already initialized'
all_data_parallel_group_ranks = []
for i in range(pipeline_model_parallel_size):
    start_rank = i * num_pipeline_model_parallel_groups
    end_rank = (i + 1) * num_pipeline_model_parallel_groups
    for j in range(tensor_model_parallel_size):
        ranks = range(start_rank + j, end_rank, tensor_model_parallel_size)
        all_data_parallel_group_ranks.append(list(ranks))
        group = torch.distributed.new_group(ranks)
        if rank in ranks:
            _DATA_PARALLEL_GROUP = group
            _DATA_PARALLEL_GLOBAL_RANKS = ranks

# Build the model-parallel groups.
global _MODEL_PARALLEL_GROUP
assert _MODEL_PARALLEL_GROUP is None, 'model parallel group is already initialized'
for i in range(data_parallel_size):
    ranks = [data_parallel_group_ranks[i]
             for data_parallel_group_ranks in all_data_parallel_group_ranks]
    group = torch.distributed.new_group(ranks)
    if rank in ranks:
        _MODEL_PARALLEL_GROUP = group

# Build the tensor model-parallel groups.
global _TENSOR_MODEL_PARALLEL_GROUP
assert _TENSOR_MODEL_PARALLEL_GROUP is None, \
    'tensor model parallel group is already initialized'
for i in range(num_tensor_model_parallel_groups):
    ranks = range(i * tensor_model_parallel_size,
                  (i + 1) * tensor_model_parallel_size)
    group = torch.distributed.new_group(ranks)
    if rank in ranks:
        _TENSOR_MODEL_PARALLEL_GROUP = group

# Build the pipeline model-parallel groups and embedding groups
# (和Build the model-parallel groups等价)
# (first and last rank in each pipeline model-parallel group).
for i in range(num_pipeline_model_parallel_groups):
    ranks = range(i, world_size, num_pipeline_model_parallel_groups)
    group = torch.distributed.new_group(ranks)
    if rank in ranks:
        _PIPELINE_MODEL_PARALLEL_GROUP = group
        _PIPELINE_GLOBAL_RANKS = ranks
    # Setup embedding group (to exchange gradients between
    # first and last stages).
    if len(ranks) > 1:
        embedding_ranks = [ranks[0], ranks[-1]]
        position_embedding_ranks = [ranks[0]]
        if pipeline_model_parallel_split_rank is not None:
            if ranks[pipeline_model_parallel_split_rank] not in embedding_ranks:
                embedding_ranks = [ranks[0],
                                   ranks[pipeline_model_parallel_split_rank],
                                   ranks[-1]]
            if ranks[pipeline_model_parallel_split_rank] not in position_embedding_ranks:
                position_embedding_ranks = [ranks[0],
                                   ranks[pipeline_model_parallel_split_rank]]
    else:
        embedding_ranks = ranks
        position_embedding_ranks = ranks

    group = torch.distributed.new_group(embedding_ranks)
```

