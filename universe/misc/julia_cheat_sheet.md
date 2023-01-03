---
title: julia小抄
author: 66RING
date: 2000-01-01
tags: 
- julia
- cheat sheet
mathjax: true
---

# julia小抄

- 从1开始索引
- 变量可用数学符号: `\alpha[tab]`
- 负数: `x = 1 + 3im`
- zero: 返回数值类型对应的0
- one: 返回数值类型对应的单位量
- 直接的分数展示: `1//3 + 1//2 = 5//6`
- 向量运算, 加`.`表示对每个元素操作: e.g. `A.^2`每个平方
- 链式比较`x > y > z`
- 对数: 2为底`log2()`, y为底`log(y, x)`
- 函数`function name(arg)    end`, 默认返回最后一行
    * 间写`name(x, y) = x + y`
    * 指定返回值: 函数名后加`::T`
- 匿名函数: `x -> x^2`
    * 多个参数: `(x, y) -> x^2 + y`
    * 函数式编程: 配合map使用
- 数组
    * 手动初始化`a = [1 2 3;3 4 5]`一行一行定义 或`a = [[1,2,3] [3,4,5]]`一列一列定义
    * 自动初始化`a = [1:3;4;4]`, `start:step:=end`
    * 指定初始化`[j^2 for i=1:3 for j=i:i+1 if i == j]`
    * `ones(type, rows, columns)`
- `::T`用于指定类型, 表示`T的实例`
- 结构体

```
# 不可修改类
struct C
    name::String
    age
end

# 可修改类
mutable struct C
    name::String
    age
end

new  = C("boo", 100)
```

## 应用

绘图

```
using Plots
plot(sin, 0, 2pi)
```

极限

```
using SymPy # 符号库
@vars x h real=true # 定义变量名称
limit(f, h, 0)  # 当表达式f中h趋于0时的极限
```

导数

```
#todo
```

定积分不定积分

```
# 求积分真实解
using SymPy # 符号库
@vars x # 定义符号
F = integrate(sin(x), x) # 求积分
res = F(b) - F(a) # 牛顿莱布尼兹公式

# TODO: 不定积分
```



