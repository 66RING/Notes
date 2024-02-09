---
title: 自动微分简记
author: 66RING
date: 2023-05-01
tags: 
- misc
mathjax: true
---

# 自动微分

> https://www.bilibili.com/video/BV1zD4y117bL/?spm_id_from=pageDriver&vd_source=fa5227c06f0a0c9f870b406f10be1d31

automatic differentiation in Machine Learning: a Survey

- 两种模式
    * 前向
    * 后向
    * 都用雅各比矩阵表示

- 自动微分:
    * 有限的基本运算是知道的, 对应的倒数也是知道的
    * 那就可以自动根据链式法则求出微分


e.g. 

```rust
f(x1, x2) = ln(x1) + x1x2 - sin(x2)
```

变为DAG有向无环图(编译原理), 方便表示

缺点: 存储中间变量


## 正向自动微分

```rust
f(x1, x2) = ln(x1) + x1x2 - sin(x2)
```

的表达式可以变成如下操作顺序, 编译原理

```rust
v-1 = x1 = 2
v0  = x2 = 5
-----------
v1 = ln(v-1)  = ln(2)
v2 = v-1 * v0 = 2*5
v3 = sin(v0)  = sin(5)
v4 = v1 + v2  = 0.693 = 10
v5 = v4 - v3  = 10.693 + 0.959
-----------
y = v5 = 11.652

// 然后逐项对输入求导
```


## 反向自动微分

> 和手动求导的过程类似

反向就是直接用输出y求输入x的倒数, 然后链式法则展开


## 雅各比矩阵

y关于x的梯度表示成一个矩阵

```
dy1/dx1, dy1/dx2, ..., dy1/dxn
dy2/dx1, dy2/dx2, ..., dy2/dxn   =   J
...
dyn/dx1, dyn/dx2, ..., dyn/dxn
```


# 具体实现

- 基于表达式
- 基于操作符重载
    * 重载`+=*/`
- 基于源码转换AST

## 基于表达式

就是手动实现add, sub, mul, div

## 操作符重载

每个计算都记录成一个`Tape`

缺点

- 额外记录Tape结构
- 不方便高阶微分
- 分支存在时难以实现

## 源码转换AST

易于分布式


# 挑战和未来

- 易用性
- 可微编程



