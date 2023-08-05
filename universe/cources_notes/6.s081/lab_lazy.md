---
title: MIT6.S081 lazy allocate lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# lazy page allocation

xv6使用`sbrk`系统调用申请内存。`sbrk`系统调用申请到内存后映射到用户空间中。当在申请大量内存时会带来很大的耗时，而且程序往往会预先申请暂时不用的内存。使用lazy allocate以提高效率，`sbrk`可以只标记以申请的虚拟地址但立刻申请物理页，当真正使用到该物理页时，产生缺页中断，然后真正分配物理内存，初始化为0, 映射。

## Eliminate allocation from sbrk()

移除`sbrk`中内存分配的功能。

sbrk系统调用返回新分配的内存的地址(即oldsizse, 即原`myproc()->sz`)。

懒分配sbrk只需增加`p->sz`然后返回oldsize即可。

测试：

- `make qemu`
- `echo hi` usertrap, panic not mapped


### 收获

- 查看`traphandler`提示信息
	* "异常"(当然syscall也是过trap的)会被`traphandler`捕获: `usertrap`, `kerneltrap`
		+ key of lazy allocation 
	* `sepc` -> pc
	* `stval` -> 虚拟地址


## Lazy allocation

在`trap.c`的traphandler中处理(用户空间的)缺页中断，然后真正分配物理页和映射。然后返回用户态。


### hints

- 判断属于缺页中断: `r_scause()`为13或15
- 可以通过`r_stval`获取`stval`寄存器，从而获得缺页的虚拟地址
- 真正物理页的分配可以参考`uvmalloc()`
	* allocate and mapping
- PGROUNDDONW "确定所需分配的页"
- 因为懒分配的策略, `uvmunmap()`页需要修改: 防止解映射未映射页
	* 需要注意的是，因为进程使用内存不一定的连续使用的，所以在懒分配的情况下真正分配的物理内存可能是连续的，所以`uvmunmap()`遍历npages时遇到为分配不能直接`break`

测试：`echo hi`执行成功


### 收获

- `traphandler`处理缺页中断
	* 通过cause和stval寄存器分别可以获得trap原因和trap的虚拟地址
- 懒分配带来"已分配内存不连续"问题，之前基于连续内存分配的设计需要考虑


## Lazytests and Usertests

压力测试通过`lazytests`和`usertests`

- 处理负数参数的`sbrk`
	* 负数参数需要缩小内存，立即使用`uvmdealloc`释放
- 考虑pagefault的`va > p->sz`的情况
- 修改fork内存复制
	* fork中调用`uvmcopy`进行拷贝，在`uvmcopy`中同样会遇到`walk`到invalid的情况，懒分配中忽视`continue`即可
- **考虑懒分配，但用户态中还没使用(分配物理页)就给系统调用使用了的情况**
	* 考虑"内核态的缺页中断"?
- 正确处理OOM问题
	* page fault`kalloc`失败应该杀死进程
- 处理栈上的page fault
	* 因为栈的从高地址向地址生长的
	* 而**xv6的栈布局和常见布局是不一样的**: 固定大小栈，栈在heap下方
		+ 所以有效的地址应该是`p->trapframe->sp =< va && va < p->sz`


### bugs

- `panic: uvmunmap: walk`
	* walk到了invalid(未分配)
	* 以及一系列"查表错误"

因为可能存在映射了但没分配的情况：

如在**二级**页表中，只用了一个pte，后面的没用到就没分配物理页，如：

```
.. 0: pte xxxx pa xxx  -> page table3
.. 1: pte xxxx pa nil  -> page table3
.. 2: pte xxxx pa nil  -> page table3
```

如果`p->sz`大于pte 0对应的虚拟地址的，那么遍历后续pte的过程中就会遇到"walk到了invalid"的情况。

同理，如是类似"映射了但没真正分配"的情况发生在**三级(最后一级)**(下一级指向物理地址)。那么就是walk到了，但是返回的`pte & PTE_V == 0`，因为没有真正分配物理页。

- `test: stacktest`
	* 会测试栈溢出的情况
	* 通过判断`sp`可以解决
	* **需要注意的是**，`r_sp()`和`p->trapframe->sp`是不一样的，因为用户态和内核态使用的栈不同，判断pagefault应该使用`p->trapframe->sp`

> To detect a user stack overflowing the allocated stack memory, xv6 places an invalid guard
> page right below the stack. If the user stack overflows and the process tries to use an address below
> the stack, the hardware will generate a page-fault exception because the mapping is not valid. A
> real-world operating system might instead automatically allocate more memory for the user stack
> when it overflows.

xv6的用户程序使用固定大小的stack(见`memlayout.h`)和figure 3.4。它和一般学习的"栈和堆相遇"的模式不同，xv6栈在堆下方，且固定大小，使用`guard page`来检测stack overflow，不会为栈动态分配内存。

`exec()`执行程序时直接使用`p->sz`的第二个PAGE做stack

> // Allocate two pages at the next page boundary.
> // Use the second as the user stack.

- syscall中访问了懒分配了的但没真正分配物理页情况

内核态中根本不会trap到`scause == 13 || scause == 15`的page fault。所以需要在用到用户空间内存的地方(如`copyin, copyout`)检测是否已经"标记懒分配(这里使用`walkadddr`判断)"和判断访问是否合法
