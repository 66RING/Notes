---
title: SGLang速览
author: 66RING
date: 2025-01-28
tags: 
- llm
mathjax: true
---

# SGLang速览

## Usage

```python
from sglang import function, system, user, assistant, gen, set_default_backend, Runtime


@function
def multi_turn_question(s, question_1, question_2):
    s += system("You are a helpful assistant.")
    s += user(question_1)
    s += assistant(gen("answer_1", max_tokens=256))
    s += user(question_2)
    s += assistant(gen("answer_2", max_tokens=256))


runtime = Runtime(model_path="meta-llama/Llama-2-7b-chat-hf")
#runtime = Runtime(model_path="mistralai/Mixtral-8x7B-Instruct-v0.1")
set_default_backend(runtime)

state = multi_turn_question.run(
    question_1="What is the capital of the United States?",
    question_2="List two local attractions.",
)

for m in state.messages():
    print(m["role"], ":", m["content"])


runtime.shutdown()
```

# Flow

- 入口api.py
- 只看解释器就够了`sglang.lang.interpreter`

## 有趣的抽象

- ProgramState
    * 跟llama.cpp的计算图执行器类似


## SglFunction

创建一个program对象, aka 生命周期, 然后后续包裹着的一些列`system`, `user`等就会往执行器里不停的添加

- QA
    * 模型是怎么启动的? 因为SglFunction的call只是把`def fun`里面的执行一遍, 似乎并没有涉及模型的启动?
        + 每个`system()`, `gen()`等调用都会触发`submit -> __execute`, 然后判断节点类型, 如果是`gen`等就会触发后端的执行

## run_program

可以把被`@function`包裹着的看成一个计算图, `run_program`就会执行这个函数从而一遍生成计算图(submit)一边执行(__execute)

## executor

### StreamExecutor

`sumbit()`后会触发`__execute`, 然后判断节点类型, 如果是`gen`等就会触发后端的执行

## 前端: SGL原语

解释器接收到一个submit后解释执行该节点

```python
    # NOTE: 被submit调用
    def _execute(self, other):
        if isinstance(other, SglConstantText):
            self._execute_fill(other.value)
        elif isinstance(other, SglGen):
            self._execute_gen(other)
        ...
```

### SglGen

调用后端执行生成, aka `self.backend.generate()`类似`model.generate()`

> self._execute_gen()


### SglConstantText

> self._execute_fill(other.value)

直接字符串拼接(prompt拼接)

### SglSelect

> self._execute_select(choices=[...])

调用后端执行选择, `self.backend.select`

定向生成, 只会返回choices中指定的选择。比对token id, 看生成的token和哪个choices最接近, 直接返回最接近的那个choices

- TODO:
    * 提供分支选择
    * 多个选最好的?
    * 示例?

### SglExprList

处理一组操作

```python
for x in other.expr_list:
    self._execute(x)
```

### SglVariable

> self._execute_variable(other)

一些原语, 如`select`会把有名变量记录下来(一些生成的结果), SglVariable拿到这些变量用于拼接prompt

### SglConcateAndAppend

> TODO: 暂时忽视


## 后端: backend/Runtime

> self.backend.generate
>
> 目前_execute_select和_execute_gen会使用到后端
>
> 模型入口: `self.model_runner.forward`

### srt

- srt是什么时候启动的
    * server.py导入就**全局启动**: `app = FastAPI()` -> `launch_server() -> uvicorn.run(app)`

- main loop?


### generate

- 调用tokenizer_manager.generate_request(obj).__anext__()
    1. 首次启动会创建一个handle loop, 不停recv detoken后的数据, 更新state, 触发完成信号
    2. handle loop只负责触发完成信号
- router process: `start_router_process`
    * 启动ModelRpcClient
    * 启动RouterManager
    * `router.loop_for_recv_requests`
    * `router.loop_for_forward`
- detokenizer process: `start_detokenizer_process`
    * 启动decoder: DetokenizerManager
    * `manager.handle_loop()`

数据流图, 以`/generate`为例

```
api call -> tokenizer -> router(model) -> detokenizer -> tokenizer -> api return
```

1. tokenizer, encode(prompt)发送给router: tokenizer -> router
2. router从tokenizer接受到请求, 保存到消息队列, 调用model, 结果发送给detokenizer： router -> detokenizer
3. detokenizer接受到router(model)处理的结果, decode, 然后发送给tokenizer: detokenizer -> tokenizer
4. tokenizer从detokenizer处理好的结果字符串, 保存结果到state后触发信号完成处理: tokenizer -> api


#### router

接受消息，记录消息队列，rpc调用模型

- `loop_for_recv_requests`
    * 接受zmq获取到的pyobj, 传递到recv_reqs
- `loop_for_forward`
    1. for req in recv_reqs
    2. model_client.step(req)

#### DetokenizerManager

> router(model) -> detokenizer -> tokenizer

- `handle_loop`
    1. `recv_from_router`, 接受model生成的input id
    2. decode
    3. 字符串send to tokenizer

#### TokenizerManager

> api call -> tokenizer -> router
>
> detokenizer -> tokenizer -> api return

- 处理API`generate_request`
    * 发送request给router
- `handle_loop`
    1. `recv_from_detokenizer`
    2. 保持结果到state
    3. 触发完成信号, http api继续执行

#### rpc client

1. 启动rpc server(model)
2. 暴露step方法来调用model的step: `self.step = async_wrap(self.model_server.exposed_step)`

#### rpc server

- 初始化
    * ModelRunner
    * radix cache
    * scheduler
- `exposed_step`
    1. `handle_generate_request`
    2. `forward_step`
        - `self.model_runner.forward`


### model runner

- `def forward`: 本质就调用model的forward
    * `forward_prefill`
    * `forward_decode`
- `load_model`
    * 定义几个case: llama, mistral什么的

### model

> e.g. LlamaForCausalLM
>
> 主要就是hijack了attn成RadixAttention

只有一点修改的llama: 张量并行qkv_proj, RadixAttention


### radix attention

> radix_attention.py

forward

```python
if input_metadata.forward_mode == ForwardMode.PREFILL:
    return self.prefill_forward(q, k, v, input_metadata)
elif input_metadata.forward_mode == ForwardMode.EXTEND:
    return self.extend_forward(q, k, v, input_metadata)
elif input_metadata.forward_mode == ForwardMode.DECODE:
    return self.decode_forward(q, k, v, input_metadata)
```

直接的page attn，主要是一个`input_metadata`的管理问题: `InputMetadata.create()`。

TODO: ?

#### triton backend
#### flashinfer backend

### generate主循环

> batch更新, forward(batch)

1. step
2. dispatch: prefil, decode, extend
3. case_forward
    - `forward_fill_batch`
        * `batch.init_extend_batch`
        * `model_runner.forward`
    - `forward_decode_batch`
        * `batch.update_for_decode`
        * `model_runner.forward`

### Batch

```python
new_batch = Batch(
    can_run_list,
    self.req_to_token_pool,
    self.token_to_kv_pool, # NOTE:大buffer: [size, key/value, head_num, head_dim] 
    self.tree_cache,
)
```

TOOD: 


### scheduler

> 优先队列管理者, 手动传入queue
>
> `new_batch = get_new_fill_batch()`。创建包含输入元数据的batch

- 执行位置: 如何出队入队的。visitor模式: 外部传入一个queue让scheduler管理
    * `handle_generate_request`向队列添加请求，交由scheduler读取读取队列信息并处理
- queue含有req信息，根据信息做调度


### forward_queue

- 入队: `ModelRpcServer::handle_generate_request -> self.forward_queue.append()`
- 出队: `for req in self.forward_queue`


### **元数据管理和整备**

> get_new_fill_batch: 便利queue, 生成batch

1. RadixCache, 分配page, TODO: cool review and learn
    * aka `generated_ids.append()`
    * tree_cache.match_prefix(input_ids)
        + `_match_prefix_helper`: 
            - dfs搜radix tree的最长匹配，否则插入新分支
            - **返回{prefix_node_list, new_node_list}**，即cache好的idx和需要计算的idx
2. scheduler更新queue: `forward_queue = scheduler(queue)`
3. 遍历搜索资源余量，调整page分配和释放
    - 搜索`can_run_list`
4. 封装batch

### RadixCache

page管理器



### TokenToKVPool

TODO: 物理page


### InputMetadata

管理PageAttention的metadata

```python
input_metadata = InputMetadata.create(
    self,
    forward_mode=ForwardMode.PREFILL,
    tp_size=self.tp_size,
    req_pool_indices=req_pool_indices,
    seq_lens=seq_lens,
    prefix_lens=prefix_lens,
    position_ids_offsets=position_ids_offsets,
    out_cache_loc=out_cache_loc,

    return_normalized_logprob=return_normalized_logprob,
)

input_metadata = InputMetadata.create(
    self,
    forward_mode=ForwardMode.DECODE,
    tp_size=self.tp_size,
    req_pool_indices=req_pool_indices,
    seq_lens=seq_lens,
    prefix_lens=prefix_lens,
    position_ids_offsets=position_ids_offsets,
    out_cache_loc=out_cache_loc,

    out_cache_cont_start=out_cache_cont_start,
    out_cache_cont_end=out_cache_cont_end,
)

input_metadata = InputMetadata.create(
    self,
    forward_mode=ForwardMode.EXTEND,
    tp_size=self.tp_size,
    req_pool_indices=req_pool_indices,
    seq_lens=seq_lens,
    prefix_lens=prefix_lens,
    position_ids_offsets=position_ids_offsets,
    out_cache_loc=out_cache_loc,

    return_normalized_logprob=return_normalized_logprob,
)

def create(
    cls,
    model_runner,           # 模型实例, 访问模型本身的配置head, kv_head, head_dim等
    tp_size,                # tp_size
    forward_mode,           # enum类型prefill, decode, extend
    req_pool_indices,       # TODO:
    seq_lens,               # NOTE:包含共享前缀在内的序列长度
    prefix_lens,            # NOTE:共享前缀的长度
    position_ids_offsets,   # TODO: NOTE:
    out_cache_loc,          # TODO:
    out_cache_cont_start=None,  # TODO:
    out_cache_cont_end=None,    # TODO:
    return_normalized_logprob=False,
):
    pass
```

- 不同case`def create()`的区别
    * Prefill
    * Decode
    * Extend: 利用共享前缀增量Prefill
    * TODO: 感觉应该先看scheduler





## 调度

TODO: 如何调度的?


## prompt信息是如何保存和传递的

TODO

> `self.text_ += comp`





