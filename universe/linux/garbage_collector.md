---
title: 垃圾回收机制原理与实现
date: 2021-09-05
tags: 
- os
- gc
mathjax: true
---

# 前言

- 动态内存分配在堆上进行
- 堆用free list管理空闲块
- 堆的每次动态分配内存是从它现有的free list中取出一块(或块中的一部分，这取决于管理free list的策略)
- 堆的free list中没有适合的块时再尝试"堆扩容"，即堆一开始并不是就拥有整个内存来分配的
- 所谓garbage就是内存中永远无法使用到的部分。这往往是由于动态申请内存后没有及时释放导致的内存泄漏。

垃圾回收就是程序能够自动将这部分无法使用到的内存free掉的过程，这个功能的执行者我们称为garbage collector(一下简称GC)。基于上述描述我们可以看一下垃圾回收机制是如何实现的。


# 垃圾回收原理

GC将heap中每个已分配的块看作一个节点。而所有已分配的块就组成了**连通图**。如果一个节点中有对其他节点的引用，我们就称这两个节点是连通的。GC的原理就是判断根节点与哪些结点连通(连通分量)，而那些没有与根节点连通的节点就是无法访问到的垃圾，可以进行free。

所谓根节点，是指不在堆中分配的数据，如栈上的函数、数据区的全局变量或者寄存器等。可以想象与根节点连通就表示节点是"可达"的，我们可以使用或正在使用。而那些不可达的就应该回收。

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/linux/garbage_collector/graph.png" alt="">


# 垃圾回收实现

了解了垃圾回收的基本原理，我们接下来就要考虑一下GC的具体实现。主要讨论垃圾回收的时机和垃圾回收算法`Mark & Sweep`

这里提供两种回收时机的思路：

- GC可以单独一个线程，周期的探测垃圾然后释放
- GC仅在申请空间，但free list无法满足要求时再触发垃圾回收机制

垃圾回收算法`Mark & Sweep`的回收时机就是上述的第二种。这样对垃圾回收的延后处理能够大幅缩小GC造成的开销。

`Mark & Sweep`分为两阶段`Mark`和`Sweep`，每次垃圾回收的触发都有执行这两个操作。

`Mark`操作首先通过`isPtr`判断指针p是不是会用到一个已分配的块，如果是返回该块的起始地址。之后将该"有用"块标记。最后遍历块中的每个数据，递归地调用`mark`函数。

```
void mark (ptr p) {
	if ((b = isPtr(p)) == NULL)
		return;
	if (blockMarked(b))
		return;
	markBlock(b);
	len = length(b);
	for (i=0; i < len; i++)
		mark(b[i]);
	return;
}
```

经过`Mark`操作后，所以用到的块都被标记了，`Sweep`操作就是简单的从根节点出发，遍历所以块，将没有标记的块释放来完成垃圾回收。

```
void sweep(ptr b, ptr end) {
	while (b < end) {
		if (blockMarked(b))
			unmarkBlock(b);
		else if (blockAllocated(b))
			free(b);
		b = nextBlock(b);
	}
	return;
}
```

这里有个很关键的函数`isPtr`。可以通过维护一个平衡二叉树实现，二叉树的每个节点都是一个已分配的块。左子树都是header地址小于父节点header地址的块，右子树都是header地址小于父节点header地址的块(header和footer描述了一个块的基本信息：大小、是否分配、标记等)。通过header中记录块大小和地址可以判断`p`使用的数据是否在块中，在则说明"可能用到"。

注意到上面说的是"可能用到"，因为p可能刚好是一个整型变量，光从数值上看与指针没有区别。因为C语言中没有专门的标记位标记这部分数据是不是指针，所以保守起见(防止漏标导致正在使用的内存被GC释放)，C语言中统统看成指针。虽然这样可能导致一些garbage标标记导致无法回收，但也比错误的释放好。


