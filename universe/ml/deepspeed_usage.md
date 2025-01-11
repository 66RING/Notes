---
title: Deepspeed多机微调使用和踩坑记录
author: 66RING
date: 2024-03-18
tags: 
- machine learning
- machine learning system
mathjax: true
---

# Deepspeed多机微调使用和踩坑记录

官方demo[DeepSpeedExamples](https://github.com/microsoft/DeepSpeedExamples)

[step1_supervised_finetuning](https://github.com/microsoft/DeepSpeedExamples/tree/master/applications/DeepSpeed-Chat/training/step1_supervised_finetuning)


## 基本使用

**NOTE: deepspeed只需要在一个节点上启动**, 它会自动使用ssh根据hostfile的内容去其他节点启动程序。[官方演示](https://www.youtube.com/watch?v=_NOk-mBwDYg&list=PLa85ZdUjfWS21mgibJ2vCvLziprjpKoW0&index=94)

多机训练主要有两个步骤:

1. 配置ssh免密登陆: 以便deepspeed能访问其他节点然后自动启动其他节点的程序
2. 安装pdsh, 如`apt install pdsh`
3. 配置deepspeed的hostfile: 以便deepspeed确定需要到哪些节点启动，使用多少张显卡

从一个最简单的程序开始

```python
# test.py
import argparse

if __name__ == "__main__":

    parser = argparse.ArgumentParser()
    parser.add_argument("--local_rank", type=int, help="Local rank. Necessary for using the torch.distributed.launch utility.")
    args = parser.parse_args()
    print(f"hello world from local rank: {args.local_rank}")
```

配置ssh免密登陆, 使用如下模板配置`~/.ssh/config`文件。其中Host字段的别名要和hostfile对应

```
Host host1
        User  root
        Hostname 172.23.244.161
        port 22
        IdentityFile ~/.ssh/id_rsa
Host host2
        User  root
        Hostname 172.23.244.165
        port 22
        IdentityFile ~/.ssh/id_rsa
Host host3
        User  root
        Hostname 172.21.137.233
        port 22
        IdentityFile ~/.ssh/id_rsa
Host host4
        User  root
        Hostname 172.21.137.234
        port 22
        IdentityFile ~/.ssh/id_rsa
```

使用`ssh-copy-id`挨个传递公钥以完成免密登陆，如`ssh-copy-id host1`。

编写`hostfile`文件, 格式如下, **第一列和ssh config中的别名对应**，`slots`表示每个节点有多少张显卡可用, 默认deepspeed会用满节点是所有slot。可以将`hostfile`文件拷贝到`/job/hostfile`, 这是deepspeed默认的hostfile路径, 也可以手动指定hostfile文件路径。

```
host1 slots=2
host2 slots=2
host3 slots=2
host4 slots=2
```

最后, 在一个节点上执行`deepspeed test.py`就可以启动, 这样的启动方式会自动使用`/job/hostfile`。也可以使用`--hostfile`手动指定hostfile文件路径: `deepspeed --hostfile ./hostfile test.py`。deepspeed会自动填充一些参数, 更多deepspeed的参数用法可以看官方教程。这里主要想表达的就要**只需要在一个节点执行**这个坑点(不像其他分布式生成要在每个机器上启动, 在所有就绪前会阻塞)。


工程实现版可以参考[DeepSpeedExamples](https://github.com/microsoft/DeepSpeedExamples), 其中[step1](https://github.com/microsoft/DeepSpeedExamples/tree/master/applications/DeepSpeed-Chat/training/step1_supervised_finetuning)就详细示范了怎么做sft。


## 通过torchrun启动

有些集群平台没有适配直接用deepspeed启动, 只适配了使用pytorch的master/worker启动。所以有必要搞清楚如何使用torchrun启动deepspeed。

原理就是挨个节点执行一下命令, 然是从环境变量中获取`LOCAL_RANK`, 因为torchrun不象deepspeed会自动添加`--local_rank`参数。其中`--node_rank`参数是必要的, 否则会一直等待。rank表示当前节点的id, local rank表示当前节点中单个显卡的id。

```bash
torchrun --nproc_per_node=$NPROC_PER_NODE --nnodes=$NNODES --node_rank=${RANK} --master_addr=${MASTER_ADDR} --master_port ${MASTER_PORT} <main.py>
```

在代码中添加获取`LOCAL_RANK`变量, 这个变量受到`--nproc_per_node`影响

```python
import os
LOCAL_RANK = int(os.environ['LOCAL_RANK'])
```

其中`$MASTER_ADDR`, `$RANK`和`$MASTER_PORT`环境变量在集群启动时会自动配好。

[PS: torchrun和torch.distributed.launch的一些区别](https://pytorch.org/docs/stable/elastic/run.html#transitioning-from-torch-distributed-launch-to-torchrun): `torch.distributed.launch`会传递`--local-rank`参数, 而`torchrun`不会。


所以总结一下用deepspeed启动和用torchrun启动的区别:

- deepspeed会根据hostfile自动启动, 默认会使用当前节点的所有显卡; torchrun需要`--nproc_per_node`指定使用多少显卡
- deepspeed在一个节点启动, 通过ssh自动启动其他节点的程序; torchrun需要手动启动每个节点的程序
- deepspeed继承了`torch.distributed.launch`的传统, 会自动给你的程序添加一个`--local_rank`参数; torchrun的`local_rank`通过`LOCAL_RANK`环境变量获取

所以可以说deepspeed就是自动在每个节点上执行了`torch.distributed.launch`, 并根据hostfile设置`--nproc_per_node`参数。


## 使用deepspeed官方代码估计所需资源

```python
from transformers import AutoModel;
from deepspeed.runtime.zero.stage3 import estimate_zero3_model_states_mem_needs_all_live;
model = AutoModel.from_pretrained("meta/Llama-2-70b-chat-hf");
estimate_zero3_model_states_mem_needs_all_live(model, num_gpus_per_node=8, num_nodes=1)
```

e.g. stage3时的llama2-70b的估计。(当然, **实测很不准**, 或许是那里没有配置正确?)

```
Estimated memory needed for params, optim states and gradients for a:
HW: Setup with 4 nodes, 1 GPU per node.
SW: Model with 68714M total params, 262M largest layer params.
  per CPU  |  per GPU |   Options
  431.97GB |   0.98GB | offload_param=cpu , offload_optimizer=cpu , zero_init=1
  431.97GB |   0.98GB | offload_param=cpu , offload_optimizer=cpu , zero_init=0
  383.97GB |  32.97GB | offload_param=none, offload_optimizer=cpu , zero_init=1
  383.97GB |  32.97GB | offload_param=none, offload_optimizer=cpu , zero_init=0
    1.46GB | 288.96GB | offload_param=none, offload_optimizer=none, zero_init=1
  383.97GB | 288.96GB | offload_param=none, offload_optimizer=none, zero_init=0
Estimated memory needed for params, optim states and gradients for a:
HW: Setup with 8 nodes, 1 GPU per node.
SW: Model with 68714M total params, 262M largest layer params.
  per CPU  |  per GPU |   Options
  215.98GB |   0.98GB | offload_param=cpu , offload_optimizer=cpu , zero_init=1
  383.97GB |   0.98GB | offload_param=cpu , offload_optimizer=cpu , zero_init=0
  191.99GB |  16.98GB | offload_param=none, offload_optimizer=cpu , zero_init=1
  383.97GB |  16.98GB | offload_param=none, offload_optimizer=cpu , zero_init=0
    1.46GB | 144.97GB | offload_param=none, offload_optimizer=none, zero_init=1
  383.97GB | 144.97GB | offload_param=none, offload_optimizer=none, zero_init=0
```

## RuntimeError: Ninja is required to load C++ extensions

[把conda的环境变量加上](https://github.com/microsoft/DeepSpeed/issues/1687)

如, 可以在代码中加上:

```python
import os
os.environ['PATH']+=':/opt/conda/bin/'
```

其中`/opt/conda/bin/`是你的cuda环境二进制程序的位置


## CPU内存OOM问题

我发了一个[issue](https://github.com/microsoft/DeepSpeed/issues/5290), 发现用的卡越多使用的CPU内存越多, 及时没开offload。单机单卡的时候CPU内存不会OOM, 当使用单机4卡启动llama2-70b时CPU比GPU先OOM了。





