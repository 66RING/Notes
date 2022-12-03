---
title: 现代cpp杂项
author: 66RING
date: 2000-01-01
tags: 
- cpp
mathjax: true
---

[TOC]

## cpp单例模式

static标记, 返回引用。static标记后构造函数就只会触发一次。

```cpp
class A {
public:
  static A &getinstance() {
    static A a;
    return a;
  }
};
```

