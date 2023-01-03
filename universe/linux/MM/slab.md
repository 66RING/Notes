---
title: slab分配器
date: 2021-01-18
tags: 
- linux
- kernel
- memory management
mathjax: true
---

## slab分配器

slab分配器的两个功能：

- 1 减少内部碎片: 小结构占用一个page不划算
    * 一般名为`kmalloc-xxx`, e.g. `kmalloc-4096`, 这些是通用型slab
- 2 作为高速缓存，存储内核中经常被分配并释放的对象

> Jeff Bonwick 发现对内核中普通对象进行初始化所需的时间超过了对其进行分配和释放所需的时间。因此他的结论是不应该将内存释放回一个全局的内存池，而是将内存保持为针对特定目而初始化的状态

当需要时内核会在直接从slab分配器的高速缓存中获取一个已经初始化了的对象，用完后不是释放它，而是放回slab分配器中。后续的内存分配不需要执行初始化函数。


### slab分配器的结构

```
cache_chain
  |
kmem_cache  ->  [ slab, slab, slab, ... ]
  |                |
kmem_cache      [ object, object, object, ... ]
```

每个cache都是一段连续的内存空间，通常使用`kmalloc()`分配。对象数目较多时不利用动态调整。因此每个cache分成了若干slabs，同一cache中的slabs储存相同的对象。一个cache用`struct kmem_cache`结构体表示

```c
struct kmem_cache{
}
```

一段缓存的所有slab分为3类，存在3个列表中：`slabs_free`(所有)对象空闲的列表，`slabs_partial`部分对象空闲的列表, `slabs_full`对象用尽的列表。

因为cache是连续的，所有每个slab都是一个连续的内存块(一个或多个连续页，通常为一页)，它们被分配成一个个对象。

一个缓存包含多个slab，一个slab包含多个对象用于分配。当一个slab中的对象分配完后会被放入`slabs_full`列表，同理一个slab可以在3个列表中移动。


### 内核函数

#### 为一个创建缓存kmem_cache_create

```c
struct kmem_cache *
kmem_cache_create(const char *name, unsigned int size, unsigned int align,
                slab_flags_t flags, void (*ctor)(void *))
```

- name：对象的名称
- size：缓存对象的大小
- align：对其的大小
- flags：分配掩码
- ctor：对象的构造函数

主要流程如下：

```c
kmem_cache_create-----------------------进行合法性检查，以及是否有现成slab描述符可用
    do_kmem_cache_create----------------将主要参数配置到slab描述符，然后将得到的描述符加入slab_caches全局链表中。
        __kmem_cache_create-------------是创建slab描述符的核心进行对齐操作，计算需要页面，对象数目，对slab着色等等操作。
            calculate_slab_order--------计算slab对象需要的大小，以及一个slab描述符需要多少page
            setup_cpu_cache-------------继续配置slab描述符
```


#### 分配slab对象kmem_cache_alloc

```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
```

- cachep：对象所在缓存
- flags：一组标志


#### 释放对象kmem_cache_free

```c
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
```

- cachep：对象所在缓存
- objp：释放的对象


#### 销毁缓存kmem_cache_destroy

```c
void kmem_cache_destroy( struct kmem_cache *cachep );
```

内核模块被卸载时执行，销毁缓存并从`cache_chain`链表上移除





