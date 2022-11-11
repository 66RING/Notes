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
