---
title: linux奇技淫巧
author: 66RING
date: 2000-01-01
tags: 
- misc
mathjax: true
---

# 记录一些"八股"tips

stack base, since 2022-12-30 14:13


## mmap and sparse array

> [post](https://offlinemark.com/2020/10/14/demand-paging/)

linux mmap的page demand, 对于匿名映射它会映射一个特殊的"zero page", 然后COW。而read的时候其实不会触发page demand, 因为假设没有write就是初始的"zero page"。

利用这点你就可以实现很大的零散数组, 它即能利用内存访问的特性又不会实际占用很多内存空间(因为大多的特殊的"zero page")。可以需要注意一些overcommit的开启情况

2022-12-30


## container_of

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

2022-09-09
