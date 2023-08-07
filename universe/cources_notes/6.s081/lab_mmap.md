---
title: MIT6.S081 mmap lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# mmap

`mmap`和`munmap`系统调用让程序可以更详细地控制他们的地址空间。

**如** 可以让内存在进程间共享，可以将文件映射到进程的地址空间，可以用于实现用户态缺页策略(如[lecture](https://pdos.csail.mit.edu/6.S081/2020/lec/l-uservm.txt)中讨论的垃圾回收算法)

这个lab你将要实现的`mmap`和`munmap`重点关注于文件到内存的映射。

可以查看`man 2 mmap`获得细节。`mmap`有多种调用方式，但这个lab只要求实现一部分功能。

```
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```

- 你可以假设`addr`永远是0，表示内核应该觉得文件**映射到虚拟内存**的哪里
- `mmap`成功返回地址或失败返回`0xffffffffffffffff`
- `length`表示映射的byte数(需要注意它可以和文件大小不同)
- **`prot`**表示映射区域的权限，可读可写可执行
	* 你可以假设`prot`是`PROT_READ`或`PROT_WRITE`或两者同时
- **`flags`**要么是`MAP_SHARED`要么就是`MAP_PRIVATE`
	* `MAP_SHARED`表示对映射区域的修改应该写回文件
	* `MAP_PRIVATE`表示不写回
	* TODO 如何理解`MAP_PRIVATE`
- `fd`是待映射的文件的文件描述符
- `offset`表示从文件的哪里开始映射，你可以认为是0

进程间对同一文件的`MAP_SHARED`映射使用不同的物理页也是可以的。

也可以实现相同文件的`MAP_SHARED`映射时共用物理页。这里实验对`MAP_SHARED`的要求仅是写回文件。所以TODO

`munmap(addr, length)`将移除指定地址范围内的映射。如果进程对该区域进行过修改，并且是`MAP_SHARED`映射，它还需要将修改写回文件中。

一个`munmap`调用只能覆盖一段已映射区域，但你可以认为它要么在开头`munmap`要么在结尾`munmap`或在对整个区域(而不是在区域中打个洞)。就是说这个实验做了简化，`munmap`不会导致vma分离成两段不连续的区间。

你应该实现`mmap`和`munmap`的大多数功能，以通过`mmaptest`测试。`mmaptest`中没有使用的`mmap`的其他功能你就不用实现。


## hints

- 将`_mmaptest`添加到makefile，然后添加`mmap`,`munmap`系统调用以通过编译
- 缺页时，页表懒加载。`mmap`不会立即申请内存读取文件，而是在`usertrap`触发缺页时。类似lab lazy
	* 之所以要lazy是为了保证大文件的`mmap`尽快执行，并且可以让`mmap`的文件大于物理内存成为可能
- 记录每个进程都映射了哪些内容。根据lecture, **定义一个表示VMA(virtual memory area)的数据结构**。
	* 记录`mmap`创建的地址，长度，权限
	* 因为**xv6内核态中没有内存分配器**，所以可以设置固定大小VMA数组，需要VMA时再从中分配
		+ 大小16就够了
- 实现`mmap`
	* 找到进程地址中未使用的内存区域，将文件映射进去
	* 将VMA添加到进程的"已映射区域表"(即每个进程维护自己的vma)
		+ VMA应该包含一个指向`struct file`的指针
		+ `mmap`会让对应文件的引用计数增加
		+ 关闭和清除时考虑引用计数，可以看看`filedup`
	* 运行`mmaptest`测试, 此时应该会缺页退出
- 添加处理映射区缺页相关的代码
	* 申请一物理页，将对应文件的4KB的内容读入
		+ 可以使用`readi`读取文件，注意需要持有锁(lab fs)
	* 映射到进程地址空间，设置权限
	* 运行测试，`mmap`阶段测试应该通过
- 实现`munmap`
	* 找到地址对应的`VMA`，取消映射(使用`uvmunmap`)
	* 如果`munmap`移除了`mmap`的所有映射(即整个文件unmap了)，让对应`struct file`的引用计数减一
	* 如果要需要映射的页有被修改过，并且是以`MAP_SHARED`映射的，要写回文件
		+ 可以参考`filewrite`
- 理想情况下你的实现只会将修改过的`MAP_SHARED`映射的页写回
	* 使用PTE的dirty bit(D)
	* 但是`mmaptest`不会检测非脏页的写回(所以你可以一股脑写回)
- TODO Modify exit to unmap the process's mapped regions as if munmap had been called.
- 修改`fork`以保证子进程有同样的以映射区域
	* **记得引用计数**
	* 当子进程缺页时，可以单独申请物理页做映射，而不用和父进程共用

测试

- `mmaptest`
	* `fork_test`
	* `mmap_test`
- `usertest`


## result

- 为何引入VMA
- 之前一致不知道内核态如何申请内存，原来xv6中内核态就没有内存分配器，需使用固定大小



- 这里mmap的`addr`参数总是0，表示映射到哪里由内核决定
	* 即找这个未使用的地址, 可以简单的使用`p->sz`后的空闲地址，然后让`p->sz`增加
- 因为是lazy map的，所以uvmunmap不检查是否已经映射。同理`uvmcopy`
- **需要特别注意offset的使用**
	* 因为fork复制了vma会**引用同一个`struct file`(refcnt++)**，如果直接使用file里的offset(`file->off`)那么一个程序对`file->off`的改变将会影响另一个程序，从而导致使用的`file->off`不合预期
	* 正确的做法可以是, **计算偏移值**: `va - start_va + vma->offset`
	* 所以现成的`filewrite`是不可用的，需要使用底层的`writei`

- 这里`mmap`和`munmap`的实现只考虑了从开始到结尾`munmap`的情况
	* 实际中可能会从vma中间开始`munmap`，这样就会导致vma分裂成两份



## TODO

- 务必写出一些深入理解再删TODO
- TODO 除了文件到内存mmap，上面提到的其他功能是否也可以实现
	* [lecture](https://pdos.csail.mit.edu/6.S081/2020/lec/l-uservm.txt)中讨论的垃圾回收算法

https://pdos.csail.mit.edu/6.S081/2020/lec/l-uservm.txt
