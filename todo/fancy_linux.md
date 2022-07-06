---
title: linux中的奇技淫巧
author: 66RING
date: datetime
tags: 
- tags
mathjax: true
---

# Abstract

一些很妙的操作，需要对计算机的深刻理解

# container_of

```c
#define container_of(ptr, type, member) ({			\
	const typeof( ((type *)0)->member ) *__mptr = (ptr);	\
	(type *)( (char *)__mptr - offsetof(type,member) );})

#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE *)0)->MEMBER)
```

使用`container_of(变量名, 变量所属结构体, 变量对应的成员名)`

解析:

1. `const typeof( ((type *)0)->member )*`获取变量类型，声明一个指针
	- **这里的0可以是任何数**，只是为了获取到它是什么类型
2. `offsetof`获取成员在结构体中的偏移
	- **这里的0是必要的**，因为本来是只能取到地址的，不如如果结构体的从0开始的，那么成员的地址就是成员的偏移
3. 成员指针减偏移就是所属结构体的地址了，再将指针类型转换然后返回

