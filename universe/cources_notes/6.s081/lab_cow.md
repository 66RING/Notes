---
title: MIT6.S081 cow lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---

# Copy on Write

虚拟内存可以通过修改标记位来让内存地址由不同含义，如通过标记PTE为只读或invalid可以触发page fault。lazy allocation和这个lab的COW都是例子。

fork会拷贝父进程的所有内容，但是这往往是浪费的，因为如果`fork`后立马`exec`执行新程序这fork花费那么大代价拷贝就浪费了。COW就是当子进程和父进程fork时先共享相同内存，当由写入操作时两个进程内存不再相同，这时才为子进程拷贝。

- fork延迟分配内存和拷贝内存
- fork仅为子进程创建页表，并引用父进程的物理页
- fork创建页表后标志父子进程的物理页(PTE)为"不可写"
	* 当尝试对这些内存写入时触发page fault
- 内核需要识别COW的page fault和lazy allocation的page fault
	* 这时父进程和子进程都共同使用origin page
	* 当有一方page fault时，为其分配新的物理页，将origin page拷贝入，然后使用新page
	* 最后将PTE修改为可写

**COW增加了释放内存的难度**: 一个物理页可能被多个程序引用。


## Implement copy-on-write

- 修改`uvmcopy()`，映射父进程的物理页到子进程，然后清除`PTE_W`标志
- 修改`usertrap()`识别page fault。COW pagefault时分配新页面，然后拷贝旧页面，最后映射新页并标记可写`PTE_W`
- **释放**: 当最后的引用释放时才释放
	* 页表中记录`reference count`
	* 分配时引用引用计数置1
	* fork时引用计数加1
	* 释放页时减1
	* `kfree()`尽在引用计数为0时将页面放回freelist
	* 可以用定长数组保存引用计数，不过需要向办法索引和选择数组大小
		+ 如可以使用`pa / 4096`来索引
		+ 数组大小可以是freelist中的最高物理地址`vm.c:kinit()`
- 修改`copyout()`使用同样的方法处理COW page fault

- 测试`cowtest`, `usertests`
- `test: simple`申请一大半物理物理内存，如果没有COW，那么fork将内存不足


### hints

- 不要受到lazy allocation影响。在非lazy allocation的情况下实现COW
- 想办法记录PTE是否COW映射，可以使用RISCV的RSW(reserved for software)位实现
- `riscv.h`提供了一些实用工具
- 当COW导致内存耗尽时，程序将退出


### result & bugs

- COW仅用修改`copyout`，因为copyout才会"write"
- 错误理解cow细节
	* 应该是最后的物理页引用origin table，而不是整个pagetable引用
	* 而且如果是整个pagetable引用，粒度也比较大，不如"写一点 COW一点"
- 对page fault的判断, 可以使用riscv页表PTE存在RSW(reserved for software)标志位
- 并发程序bug，操作加锁的粒度问题
	* 具体看下一章说明


### **并发程序与data race**导致的bug

观察下面的程序：这时发送COW时拷贝到新物理页的方法。其中`rc_value()`和`kfree()`内部都会通过加锁判断引用计数。

```c
if(rc_value(pa) <= 1) {
	*pte = PA2PTE(pa) | flags;
} else {
	if((mem = kalloc()) == 0) {
		return -1;
	}
	memmove(mem, (char*)pa, PGSIZE);
	*pte = PA2PTE(mem) | flags;
	kfree((void*)pa);
}
```

锁模型大致如下：

```
Lock(rc)
rc_value 检查rc值
Unlock(rc)

如果rc不为1 {
	Lock(rc)
	kalloc, 某rc[pa_new]++
	Unlock(rc)
	拷贝到新物理页

	Lock(rc)
	if --rc[pa_old]==0 , kfree
	Unlock(rc)
}
```

考虑2个进程同时进入"rc不为1"的case。这两个进程进入"rc不为1"这个case中会数据拷贝到新物理页，然后释放"原物理页"。此时如果又`fork`子进程，那么子进程引用的"原物理页"就被释放了。因此需要这里的**"释放原物理页操作"和`uvmcopy`的"引用原物理页操作"需要互斥**。

所以，如果整"判断rc值，然后cow"操作加锁，则在操作中每次rc只会变化1，且原物理页是保存的，那么就避免了"原物理页被提前释放"问题

但是如果`rc_lock`不是在`kalloc/kfree`内部，那么需要在所有用到`kalloc/kfree`的地方多加考虑。并且不要忘了小心"操作不可分"问题。

综上，这里将讨论技术细节：

1. 可以在`kfree`内部对rc加锁，而`kalloc`不需要加锁，因为从freelist申请到的pa是唯一的，对于同一rc项不会存在数据竞争。而且`kalloc`后都会初始化rc为1
2. 不需要在"rc!=1"的case中kfree，只需要`rc_decrease`。因为rc不为1一定不会释放
3. 除`kfree`内部需要加锁保护外，其他地方需要的就只有: "uvmcopy时"和""

- 这里问题的关键是**原物理页被提前释放**
- 可以尽量采取**两段锁协议**来防止不正确行为的发生。需要多小心什么操作是不可分的
- 锁怎么设计好难，如果kalloc要修改rc值，那么每个用到分配的地方都要`rc_lock`? 如果在`kalloc/kfree`内部封装又会让锁的粒度变小从而"违反两段锁"导致上述问题。





