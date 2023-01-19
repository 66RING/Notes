---
title: rust杂项
author: 66RING
date: 2000-01-01
tags: 
- rust
mathjax: true
---

# Abstract


# Preface


# Overview

- `NonNull<T>`
    * 创建的时候使用`*mut T`创建: `NonNull::new(*mut T)`
    * 假设说包裹的指针非空
- `mem::forget()`

## 小工具

### 火焰图

```
$ cargo install flamegraph
$ cargo framegraph
```


### 查看asm

输出rust和对应汇编代码

```
$ cargo install cargo-asm
$ cargo asm --rust path::to::a::function
```


## Cell vs RefCell vs Cell

- 都可以在没有所有权, 没有可变引用的情况下(即进有不可变引用)做内部可变
- 但不是都能通过不可变引用`&T`拿到内部数据的`&mut`
    * Cell, RefCell都没有办法在{不获取所有权/非可变引用}的情况下获取到&mut的
        + 而很多情况我们只能拿到不可变引用且拿不到所有权: 如静态变量
        + 而UnsafeCell可以通过get()获取到裸指针然后转换成&mut




