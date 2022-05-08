---
title: A fork() in the road论文笔记
author: 66RING
date: 2022-05-07
tags: 
- papers
mathjax: true
---

# Abstract

> fork()对于现代程序员来说是一个糟糕的抽象
>
> 别再使用fork()了，可以使用更现代的`posix_spawn()`
>
> 是时代的错误，与现代系统格格不入

- 不是线程安全的
- 不可扩展(unscalable)
- 不再简易

万恶之源: fork管得太多了

1. fork即同时插手了进程和它的地址空间
	a. 这就导致从IO buffer到kernel bypass网络的一些实现变得tricky。("你以为bypass了，但fork偷偷做了")
2. fork不是模块化的(don's compose)
	- (像第一点说的，它涉及太多东西)，那么用户使用fork的时候都得考虑
	- 维护不提供一个模块化的库?
3. 阻碍创新
	- 模块化的自由组合 vs 固定一套的fork流程


# 正文内容

## 为何Fork

历史原因：fork通用，不用过多改变


## 现代人眼中的Fork

- 不再简单
	* 关得太多，copy得太多：file lock, timer, async, tracing, etc.
	* 管得多要么套路固定死，要么一堆参数提供定制化
- *don't compoes*
	* 经典刷新IO buffer避免重复输出问题，就是因为它put together那就很难处理好
	```c
	int main() {
		int i;
		for(i=0;i<2;i++) {
			fork();
			printf("-");
		}
		return 0;
	}
	```
- 不是线程安全的
- 不安全
	* 子进程是父的copy，那就知道父进程的状态
- 慢
	* 毕竟它任务多
	* 虽然有COW技术改善
- 不可扩展
	* "不可扩展问题源自: 共享"，有[共享就有瓶颈](https://github.com/66RING/Notes/tree/master/universe/os/noscalable_lock_and_solution.md)
- memory overcommit: fork实现的选择问题
	* 保守的做法，能容纳整个内存的拷贝fork(并COW)才成功。显然对大进程或者永远不修改的进程保守的实现不好
	* 激进的做法，只映射不管够(overcommit)，后头有虚拟内存顶着(换入换出，所以redis建议关闭overcommit)
	* 虽然unix不支持overcommit，但是广义上来说fork的实现会带来如上问题，那就得小心了


## 实现fork(实现fork会遇到的问题)

1. fork与单一地址空间不兼容
	- 许多现代很多"单一地址空间执行", e.g. unkernel
		* 如Drawbridge就是`CreateProcess()`将二进制加载到另一个分区
		* 虽然欠安全欠隔离，但是这样就可以在一个闭包(enclave)里运行window环境了
		* 而fork如果要实现这个，就要考虑复杂的链接和比较(因为零散)
2. fork与各种各样的硬件不兼容
	- 怎么在硬件的地址空间中拷贝进程状态
3. fork影响整个系统
	- 每一层抽象都要考虑到fork(fork-based)


## 弃用fork吧

- 新的抽象：Spawn
	* `posix_spawn()`的主要缺点是：不能替换一些fork exec的场合
		+ 如设置终端属性, e.g.`set 3>-`和切换命名空间
		+ 不过都是可以改的
- 使用`vfork`替代
	* 共享父进程的地址空间，知道子进程调用exec
- COW


# 总结与收获

- 管得太多就要注意了
- "通用", "市场占有率", "隐患"
- overcommit很多细节要考虑的: 具体实现的选择带来的隐患
	* "隐式bypass buffer", 即虚拟内存偷偷用到磁盘

