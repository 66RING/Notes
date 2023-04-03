---
title: MIT6.S081 thread lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# Multithreading

熟悉多线程，实现用户空间线程间切换，使用多线程编程，实现**barrier**。


## Uthread: switching between threads

实现一种用户空间的上下文切换机制。

修改`user/uthread.c`和`user/unthread_switch.S`和在makefile中添加uthread程序。`uthread.c`中包含了测试和框架，还需要实现线程创建和线程切换。

设计一个线程切换方案，保存上下文，切换和恢复。测试程序：`uthread`。

- 在`user/uthread.c`中添加`thread_create()`和`thread_schedule`
	* `thread_schedule`一开始能运行一个执行线程，线程在自己的栈上执行传递到`thread_create()`的执行函数
		+ 可以通过将`sp`置为`t->stakc`实现
- 在`user/uthread_switch.S`中添加`thread_switch`
	* `thread_switch`保存寄存器和恢复寄存器。需要自定义怎么保存(数据结构)
	* 在`thread_schedule`中调用`thread_switch`


### hints

- `thread_switch` needs to save/restore only the callee-save registers. Why?
```
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```
- 可以通过`uthread.asm`调试
- 调试方法:
```
(gdb) file user/_uthread
Reading symbols from user/_uthread...
(gdb) b uthread.c:60
```

### result

需要注意**栈的生长方向**，因此在设置线程栈时应该将"栈高地址"传入: `sp = &t.stach[SIZE-1]`，而不是`sp = &t.stack`

- 线程起始也不神秘嘛
	* 资源相对独立：kalloc自己的栈
	* 和进程共用页表：就不用模式切换和页表切换，也不用情况i-cache指令缓存
	* 同样的上下文切换逻辑


## Using threads

在一个实机上探索并行编程，使用多线程和锁来操作哈希表。使用`pthread`库，可以查看`man pthreads`。`notxv6/ph.c`包含了一个简单的哈希表，尝试多线程正确的完成。

`ph`程序首先通过`put()`函数插入许多key，然后输出puts per second，同样的然后是`get()`的benchmark。

`ph`程序能够测试出put了但是missing的数量(因为多线程共享资源出现了覆盖的情况)。使用加锁防止这种情况发生。

一些用法示例：

```c
pthread_mutex_t lock;            // declare a lock
pthread_mutex_init(&lock, NULL); // initialize the lock
pthread_mutex_lock(&lock);       // acquire lock
pthread_mutex_unlock(&lock);     // release lock
```

为了提速，可以尝试使用"lock per hash bucket"

测试：`make grade`通过`ph_safe`测试


## Barrier

实现[barrier](http://en.wikipedia.org/wiki/Barrier_(computer_science)): 特定线程需要等待直到其他所有线程达到了某个阶段。使用pthread的**条件变量**。

`notxv6/barrier.c`中包含了不完整的barrier。每个线程执行一个循环，循环中线程调用`barrier()`会sleep随机时间。要求每个线程等在`barrier()`中，知道所有nthread个线程都调用了`barrier()`。

```c
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

- **注意**, `pthread_cond_wait(&cond, &mutex)`会释放锁, **因为假设是已经lock的**，然后sleep，苏醒时申请。
- `pthread_cond_broadcast()` call unblocks all threads currently blocked

- 使用`struct barrier`
- 每个回合`bstate.round`：所有线程调用`barrier()`到所以线程离开。
	* 需要小心怎么判断所以线程离开，因为一个线程离开后可能立刻又调用

测试`make barrier`, `./barrier 2`不应该有异常。


### result

- `pthread_cond_wait`, `pthread_cond_broadcast`的用法
	* 条件变量可以简化很多操作
	* `pthread_cond_broadcast`会唤起所以因`pthread_cond_wait`进入休眠的进程。但是他们唤醒后得竞争锁
- 深刻理解条件变量模型

