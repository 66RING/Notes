---
title: Buddy系统的原理及实现
date: 2021-01-19
tags: 
- linux
- kernel
- mm
mathjax: true
---

# Buddy系统的原理及实现

> 本质上就是一个多级链表, 然后使用伙伴系统这种特殊的"索引方式"开快速分配和查找所需内存

通过"二分查找"的方式快速找到所需大小的内存空间。对于大内存的分配是比较快的。

基本要素:

1. 多级freelist用于索引: e.g. 有大小为32, 64, 128...的freelist
2. heap空间用于分配, bitmap用于标识是否分配
3. 对齐空间内存布局用于快速删该

第3点情况举个例子如下:

```
addr: 5120 ------------- align(所属freelist类型, 如512)
struct {
    T value;
    Node* prev;
    Node* next;
}
```

- 分配时: 将这个地址(`5120`)返回, 存在这个地址的节点结构体覆盖掉
- 回收时: 再这个地址分配节点结构体, 然后接到freelist链表中

在介绍buddy系统前先做个铺垫：空闲链表和内存池。

- 大heap待分配 + bitmap表示分配情况
- 多组空闲列表管理buddy分割后的内存, 方便快速查找

- Q: 释放后在freelist上的组织情况
    * 即怎么高效实现合并: 需要删除链表上旧的项 -> ⭐ 可以物理地址头部就是节点的内存布局!!

https://zhuanlan.zhihu.com/p/73562347


## 空闲列表

内存中的空闲内存块以列表的形式组织起来，称为free list。需要内存分配时扫描列表的空闲内存块，从空闲列表中移除，释放时再放回。这里就会存在不同的内存块选择策略：

- First fir
    * 取最先遇到的大小足够的块
- Best fit
    * 取和申请内存大小最接近的
- Worst fit
    * 取和申请内存大小最不接近的


## 内存池

伙伴系统根据块的大小分类空闲列表, 每个空闲列表中的空闲内存块大小的一样的。(如32, 64, 128字节)

```
| 32 | -> | 32 | -> | 32 |
  |                    
| 64 | -> | 64 | -> | 64 |
  |                    
| 128| -> | 128| -> | 128|
  |
```

每次分配内存从大小最接近(最小大于)的空闲列表中分配。如果列表没有空闲内存可以分配，则从更大一级的空闲列表中分裂，然后分配。以分配128大小的请求为例。

当块大小为128的空闲列表没有内存可以分配时，则查找块大小为256的空闲列表。一个256的空闲链表分裂成两个128, 一个128用于分配，另一个128则插入128空闲列表中。更新bitmap

## 释放与合并

释放内存时，如果没有相邻的空闲内存，则将释放的内存块放入其对应大小的空闲列表。如果有相邻，则把相邻的内存合并成新的大内存，放入对应大小的空闲列表。

释放一块内存时(假设类型为128), 查看其相邻128的内存块是否空闲(向上找向下找依具体情况而定), 如果相邻128块空闲，**且两块组合后256对齐**，则可以合并成大小为256的块然后插入256的空闲链表中。

因为伙伴系统这种内存对齐的机制，链表上的节点的地址可以通过链表类型计算出来，从而可以方便的从旧链表中删除，然后插入新链表。e.g. 释放一个地址1280, 大小为128

根据伙伴系统的内存布局, 通过下面这种方式就可以计算出链表节点的位置。

```
Node* ptr = ROUNDDOWN128(pa);
```


## Q & A

- 什么时候需要连续的物理内存?
    * 设备看到的是物理内存, DMA。内核需要访问物理内存来与设备通信, 这样一来有些传输就需要用到大块的连续物理内存
- 连续物理内存不是通过页表解决吗? 为什么还要修改分配器
    * 是通过页表解决, 但是要通过页表解决前得要先有物理内存来分配啊。就比如说页表映射的0～9的PA, 那你下一步映射要找到10~19才行，所以需要一种能够快速查找的分配器