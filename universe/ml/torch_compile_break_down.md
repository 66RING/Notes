---
title: Torch Compile解析
author: 66RING
date: 2025-11-25
tags: 
- pytorch
- compiler
mathjax: true
---

# Torch Compile解析

> 二次加工 from: https://mp.weixin.qq.com/s?__biz=MzYyNTg1OTA5MQ==&mid=2247484015&idx=1&sn=1606439595d5049076c4c7664f4811bc&chksm=f0208d13c7570405a915b78ab21b3a2d3a0c54db1e838ea8df4fa3afccf1bc9c29873cc1a606&cur_album_id=4233444225359986690&scene=190#rd

commit: 69b05913fb0332f9a938c74e26b106e2bd24d82e

1. **TorchDynamo**: 字节码分析 + 图捕获
    - 生成FX Graph(逻辑的, 语义表示)
2. **AOT Autograd** (joint-graph): 前后向联合优化
    - 前向反向联合表示, 从而能做全局(fwd+bwd)的优化
3. **TorchInductor**: 分解 + 融合 + 调度 + 代码生成
    - 生成IR(具体的, 操作的实现), 实际操作的优化
4. Runtime: 守卫检查 + 缓存管理


## 背景

- 算子融合
    * 优化io和launch时间
- 融合的限制
    * 可融合
        + elementwise(pointwise)可以直接融合
        + reduction(规约)根据规约模式不同融合模式有区别
    * 开销(trade-off)
        + 寄存器压力 vs 融合粒度
- AOT(Ahead-Of-Time Autograd) -> 针对训练场景的优化
    * 背景: autograd机制
        + fwd: 保存激活值
        + bwd: 根据激活值计算梯度
    * 问题:
        + 需要保存什么, 何时不再使用再单个fwd/bwd中是不知道的。优化器需要看到全局才能最优
    * 解决方案 ： AOT
        + 编译期生成fwd + bwd的计算图(joint-graph), 优化器就有了全局的视角
    * 好处:
        + 重计算感知和自动重计算
        + fwd, bwd融合
        + 内存布局优化: 减少拷贝, 申请释放等内存操作的开销

## 编译流水线

四个核心阶段

1. **TorchDynamo**: 字节码分析 + 图捕获
2. **AOT Autograd** (joint-graph): 前后向联合优化
3. **TorchInductor**: 分解 + 融合 + 调度 + 代码生成
4. Runtime: 守卫检查 + 缓存管理

```
Python → FX → Joint → IR → Kernel

Python 函数
    ↓
[TorchDynamo] 字节码分析 + 图捕获
    ↓
FX Graph + Guards
    ↓
[AOT Autograd] 前后向联合（训练时）
    ↓
Joint Graph → Forward Graph + Backward Graph
    ↓
[IR 转换] FX → Core ATen → Prims
    ↓
[TorchInductor] 分解 + 融合 + 调度 + 代码生成
    ↓
设备内核 (Triton/C++)
    ↓
[Runtime] 守卫检查 + 缓存管理
    ↓
执行
```

1. Python 调用 model(x)
2. TorchDynamo 拦截，分析字节码，构建 FX Graph，生成守卫
3. AOT Autograd 处理（如果是训练模式）
4. Inductor 分解算子、融合、生成 Triton 代码
5. 调用 Triton/NVCC 编译器生成机器码
6. 执行编译后的 kernel
7. 返回结果

功能解析


| 层次          | 输入               | 输出                             | 核心职责             | 关键约束                |
|---------------|--------------------|----------------------------------|----------------------|-------------------------|
| TorchDynamo   | Python 字节码      | FX Graph + Guards                | 捕获纯张量计算       | 遇到不支持构造时断图    |
| AOT Autograd  | FX Graph (forward) | Joint Graph / Forward + Backward | 前后向联合优化       | 只在训练时启用          |
| IR 转换       | FX Graph           | Core ATen IR / Prims IR          | 标准化表示           | 函数式语义，无 in-place |
| TorchInductor | Core ATen IR       | Triton/C++ 代码                  | 融合、调度、代码生成 | 特定硬件的优化策略      |
| Runtime       | 编译后的函数       | 执行结果                         | 守卫检查、缓存管理   | 维护正确性保证          |


## TorchDynamo(FX Graph)

> 从python提取计算图生成FX Graph

- FX Graph构建
    * 符号执行: 不真正的执行, "相当于符号推导"
    * 处理控制流: 静态展开 + 运行时展开
    * 守卫生成: "加assert", 记录类型、形状等
        + 守卫检查会被编译成一个高效的函数
    * 断图策略: 如有数据依赖等场景需要运行时决定
        + 控制流: e.g. if -> 拆分两部分
        + 不支持的python操作: e.g. print
        + 外部库
        + 使用`fullgraph=True`可以在遇到段图时报错
        + 使用`TORCH_LOGS=graph_breaks`环境变量可以显示段图原因和位置

### FX Graph

> TorchDynamo的产物
>
> FX: 用python对象描述图, 而不是其他抽象/库

- Proxy + FakeTensor来捕获所有操作信息(操作对象, 操作内容)
    * 后续优化可以参考这些信息
- FX Graph变换
    * 节点增删改
    * PASS优化

## AOT Autograd

> Joint Graph: fwd, bwd联合优化

e.g. 构建joint graph: fwd, bwd联合在一起

```python
def joint(x, weight, grad_h2):
    # === Forward ===
    h1 = torch.matmul(x, weight)
    h2 = h1.relu()
    
    # === Backward ===
    # 注意：后向依赖前向的某些中间结果
    grad_h1 = grad_h2.clone()
    grad_h1[h1 <= 0] = 0
    
    grad_x = torch.matmul(grad_h1, weight.t())
    grad_weight = torch.matmul(x.t(), grad_h1)
    
    # 返回：前向输出 + 梯度
    return h2, grad_x, grad_weight
```

- 优化内容:
    * min-cut分区: 生命周期, 较少saved tensor
    * 重计算 vs 保存 tradeoff
    * 压缩保存: relu用bitmap保存
    * ...
- 效果收益:
    * 更少的activation占用 -> 可以更大batch
    * 提升GPU利用率


## TorchInductor

> IR, PASS和代码生成
>
> ATenIR生成目标代码


### 算子融合

Inductor把算子分成三类做融合

- Pointwise（逐点）：输出的每个元素只依赖对应位置的输入
    * 例子：add, mul, relu, exp, tanh
    * 融合策略：直接串联
- Reduction（规约）：输出元素依赖多个输入元素
    * 例子：sum, softmax, layer_norm
    * 融合策略：persistent reduction（把 reduction 和后续 pointwise 融合）
- Template（模板）：复杂的结构化计算，有专门的实现
    * 例子：matmul, conv2d（调用 cuBLAS/cuDNN）
    * 融合策略：通常不融合，直接调用库

### 调度器

> 资源如何分配

一些关键考虑

- Block size：每个线程块处理多少元素
    * 太小：并行度不足
    * 太大：寄存器压力过大，occupancy 下降
    - 典型值：256-1024
- Tiling：如何分块访问数据
    * 目标：最大化 L1/L2 cache 命中率
    * 对于大 tensor，需要分块处理
- 向量化：一次加载多少元素
    * GPU 内存访问是按 32/64/128 bytes 对齐的
    * 合并访问可以提高带宽利用率


### 模板系统

> Inductor 使用 Jinja2 模板生成代码

e.g.

```python
triton_template = """
@triton.jit
def {{kernel_name}}({{params}}):
    pid = tl.program_id(0)
    offsets = pid * {{BLOCK_SIZE}} + tl.arange(0, {{BLOCK_SIZE}})
    mask = offsets < {{n_elements}}
    
    {% for load in loads %}
    {{load.name}} = tl.load({{load.ptr}} + offsets, mask=mask)
    {% endfor %}
    
    {% for op in ops %}
    {{op}}
    {% endfor %}
    
    tl.store({{output_ptr}} + offsets, {{output_var}}, mask=mask)
"""
```

## Runtime

- 守卫检查
- 变体缓存
- 重编译和泛化

## 三段式优化

1. FX Graph层做python级别的算子优化
    - 规范化, 联合优化
2. Inductor IR级别的优化: 计算模式, 内存布局
    - **scheduler**
3. 代码生成优化: 具体设备相关优化


### 阶段1: FX Graph Passes(Lowering前)

> 操作Aten/Prims ops
>
> Lowering前

五个阶段

1. Pre-Grad Passes
    - 结构规范化(消除split/cat)
    - 形状传播
    - padding调整
2. AOT Autograd
    - fwd+bwd联合追踪
3. Joint Graph Passes
    - Peephole 优化（常量折叠、冗余视图消除） 
    - 随机数处理
4. Min-Cut Partitioning
    - 拆分为 fw_graph + bw_graph 
5. Post-Grad Passes
    - 局部性重排（reorder_for_locality）
    - No-op 消除(e.g. 两次相同的transpose)
    - Reinplace（功能化→原地）

#### Pre-Grad Passes

> 只能看到前向

- 不改变图结构, 加metadata
- 规范化
    * e.g. 防止split/cat等在转换成IR后信息丢失
- 消除冗余
    * e.g. 减少处理的节点数



#### AOT + Joint Graph

> 能看到前向和后向

- AOT Autograd生成后续图
- fwd, bwd两个图合并
- Peephole 优化
    * "通过一个小孔看代码", 只关注局部模式
    * e.g. `y = x * 1.0`优化成`y = x`
- 继续消除冗余操作等...


#### Min-Cut Partitioning拆分

拆分Joint Graph成fwd, bwd

- fwd包含fwd和saved tensor
- bwd包含saved tensor
- 做一些能看到全图的优化

#### Post-Grad Passes

fwd, bwd已拆分。分别优化前后向图

1. 调整节点顺序, 局部性重排
    - 优化cache, 生命周期等
2. 消除no-op: e.g. 连续两次相同的transpose
3. 用inplace优化避免新建tensor操作的开销
4. 其他: 死代码, 分布式优化..
    - Lowering前最后一个阶段, 所有多点优化


### 阶段2: Inductor IR Passes(Lowering后, codegen前)

> Lower fx graph到IR
>
> FX Graph -> torch/_inductor/lowering.py -> IR

遍历FX Graph, 查表生成IR

```python
# pytorch/torch/_inductor/lowering.py:115
lowerings: Dict[OpOverload, Callable] = {}

# ... register_foreach_pointwise
```

FX Graph表示了语义但是不知道具体应该怎么执行(e.g. 怎么加, 调库还是什么)。lower成Inductor IR后就知道了具体的操作:

```python
# FX Graph: 知道语义, add和两个参数x, y
%add = call_function[target=torch.ops.aten.add.default](
    args=(%x, %y)
)

# Inductor IR: 知道具体操作, 计算模式
Pointwise(
    device=torch.device('cuda'),
    dtype=torch.float32,
    inner_fn=lambda idx: ops.add(
        x.make_loader()(idx), 
        y.make_loader()(idx)
    ),
    ranges=[SymInt(100), SymInt(200)]
)
```

Pointwise可以看成是Inductor IR对计算模式的抽象, scheduler根据这个信息(类型, 范围, 访存模式等)可以判断是否可以融合。


#### Lowering

1. 查表: Lowering Registry
    - 注册所有aten算子, e.g. `@register_lowering(aten.add)`
        * e.g. pointwise算子只需要循环 + 循环体(add, mul, div...)
2. Lowering方法: 算子分类pointwise, reduction, template
    - Pointwise: 循环 + 循环体(op)
    - Reduction: 循环(外循环) + 规约维度 + 规约方式(op)
    - ExternKernel(调用外部库)

#### 内存布局决策

> Layout 的类型（pytorch/torch/_inductor/ir.py:3882）
>
> Lowering时要根据FX Graph记录的逻辑形状决定物理布局
>
> 复用逻辑视图(但引入额外索引计算), 还是物化视图(但需要额外拷贝)


本质就是shape + stride。主要有两个抽象: `FlexibleLayout`(动态布局), `FixedLayout`(静态布局)。scheduler可以修改动态布局的信息, 供后续使用

e.g. reshape, tranpose等可以只改变布局信息

**什么时候会物化（materialize）？** (1)当多次view被使用时, 避免重复的索引计算。(2) 无法处理的复杂stride时(非连续)

**物化的实现(生成一个copy kernel)**: `物化的实现（pytorch/torch/_inductor/ir.py:8320）`

#### IR结构

IR表示

```python
class IRNode:
    def get_size(self) -> List[Expr]:
        # 返回输出的形状
        pass
    
    def get_stride(self) -> List[Expr]:
        # 返回输出的 stride
        pass

    def realize(self) -> Buffer:
        # 物化：将计算转换成实际的 Buffer
        pass
    
    def make_loader(self) -> Callable:
        # 返回一个 loader 函数，用于读取这个节点的输出
        pass
```

存储和视图

```python
class TensorBox:
    # 顶层抽象，表示一个 tensor
    # 内部包含一个 StorageBox
    data: StorageBox

class StorageBox:
    # 可变存储，支持原地操作
    data: IRNode  # 可能是 View、Pointwise、Reduction 等
    
    def realize(self):
        # 物化：将 self.data 转换成 ComputedBuffer
        # 触发条件：
        # - 被多次使用
        # - 读取次数超过阈值（config.realize_acc_reads_threshold）
        # - 需要明确的内存分配
        pass

class Buffer(IRNode):
    # 已分配内存的 tensor
    name: str
    layout: Layout

class InputBuffer(Buffer):
    # 输入参数（来自 FX Graph 的 placeholder）
    pass

class ComputedBuffer(Buffer):
    # 计算结果（来自 Pointwise、Reduction 的物化）
    data: Loops  # 包含循环体代码
```


### 阶段3: Codegen Passes

> 根据目标设备生成目标代码
>
> 设备相关的关键优化参数: block size, tiling, avx等
>
> IR -> torch/_inductor/codegen/ -> code(str)

Codegen 接收这个 FusedSchedulerNode，生成 Triton Python 代码：

```python
# Scheduler 输出
fused_node = FusedSchedulerNode(
    nodes=[pow_node, mul1_node, add1_node, ...],  # 8 个融合的节点
    device='cuda',
    group=(device, normalized_size)
)

# Codegen 输出（Triton 代码）
@triton_heuristics.pointwise(
    size_hints={'x': 4096}, 
    filename=__file__,
    triton_meta={'signature': {...}, 'device': DeviceProperties(...)},
)
@triton.jit
def triton_poi_fused_gelu_0(in_ptr0, out_ptr0, xnumel, XBLOCK : tl.constexpr):
    xoffsets = tl.program_id(0) * XBLOCK + tl.arange(0, XBLOCK)
    xmask = xoffsets < xnumel
    x0 = xoffsets
    tmp0 = tl.load(in_ptr0 + x0, xmask)
    # ... 融合的计算 ...
    tl.store(out_ptr0 + x0, tmp8, xmask)
```

#### Triton代码结构

1. 装饰器(Heuristics)
    - 大小, 类型, 怎么tune
2. triton jit
3. kernel主体

- block size选择
    * sm利用率
- warp数
- autotune

#### 工程细节

- 装饰器选择
    装饰器的选择取决于 kernel 的特征（torch/_inductor/codegen/triton.py:5036）。
- 输入输出缓存(triton代码生成在哪)
    * 文件路径（torch/_inductor/codecache.py:1844）
        + 默认路径`/tmp/torchinductor_<username>/...`
    * 命名策略（torch/_inductor/config.py:1384）
        + e.g. `TORCHINDUCTOR_UNIQUE_KERNEL_NAMES=1`可以关闭唯一命名
    * triton自己的缓存
        + `~/.triton/cache`

#### 调试与日志

> 如何查看生成的 Triton 代码？

- 方法1: `export TORCH_LOGS=output_code`
- 方法2：保留编译产物`export TORCHINDUCTOR_CACHE_DIR=/path/to/cache`
- 方法3：Dump IR`torch._dynamo.config.output_code = True`


## Scheduler

> 融合的决策: 资源(寄存器)压力和性能的tradeoff

Scheduler决策的依据: 全融合(寄存器压力大)或部分融合

- Tensor 的大小
- GPU 的寄存器数量
- 中间结果是否被多次使用
- 启发式的代价模型

### 依赖分析

- 必须保持的依赖(MemoryDep)
    * RAW(Read After Write)
- 某种情况可以打破的依赖(WeakDep)
    * WAR(Write After Read, 反依赖)
    * WAW(Write After Write, 输出依赖)

之后就是循环依赖检查, 拓扑排序


### 融合决策

遵循一套规则

1. 相同的迭代空间
    - e.g. pointwise算子
2. 兼容的设备和dtype
3. 依赖允许
4. 启发式规则
    - 小节点优先融合: kernel开销大于计算本身
    - 共享数据融合: 共享一次读
    - 寄存器压力评估
    - 基准驱动(现场跑个benchmark测试)
        * `torch/_inductor/scheduler.py:6168`



