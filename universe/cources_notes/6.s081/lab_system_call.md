---
title: MIT6.S081 syscall lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# system calls

- user space
	* `user.h`
	* `usys.pl`
- kernel space
	* `syscall.h`
	* `syscall.c`
- process related
	* `proc.h`
	* `proc.c`


## trace

实现trace，**以便后续实验调试**

trace系统调用

- 一个int参数, "mask"
	* `trace(1 << SYS_fork)`
- 对tracing的系统调用，打印返回值
	* `<pid>: syscall <name> -> <return value>`
- 效果只影响使用的进程和其子进程


### hint

- 添加makefile
- 添加syscall
	* 用户空间添加syscall，`user.h`, `usys.pl`
		+ makefile使用perl脚本`usys.pl`生成真正的`usys.S`
	* 内核空间添加`kernel/syscall.h`
- 实现syscall本call
	* `kernel/sysproc.c:sys_trace()`
	* 修改`proc`结构，记录mask, 修改`fork()`(`kernel/proc.c`)，使能继承mask
	* 添加终端向量表，trace中断码, 中断服务程序等, `kernel/proc.h`, `kernel/syscall.c`,  `kernel/sysproc.c:sys_trace()`
	* 修改`kernel/syscall.c:syscall()`函数，每次系统调用返回时检查trace标记，并打印
	* 运行`trace 2 usertests forkforkfork`测试


### 收获

- xv6中，系统调用获取参数的方法`argint()`等
	* 为何要这么: 约定俗成参数放在`trapframe`中的`a0~a5`寄存器中
- 体会用户空间和内核空间
	* 为何用户空间的"syscall"和内核空间的syscall有什么区别: 用户空间需要调用汇编指令`ecall`陷入内核，通过封装成"用户空间的syscall"提供便利。陷入内核后会通过`trapframe`的信息调用相应的syscall
- 熟悉xv6目录结构
	* `kernel/proc.c`提供一些内核态程序主函数，如`fork`, `kill`等
	* `kernel/proc.h`维护cpu寄存器, proc结构, trampline等
	* `kernel/sysproc.c`提供系统调用处理函数，如`sys_`开头的函数


## sysinfo

- 一个`struct sysinfo`指针作参数接收返回值(fill it)
	* `freemem`表示内存剩余字节数
	* `nproc`表示`state != UNUSED`的进程
- 测试`sysinfotest`


### hint

- read in kernel, copy back to user(see `sys_fstat()->filestat()->copyout()`)
- 计算空余内存: 在`kernel/kalloc.c`中添加获取空闲内存的进程数
- 计算进程数: 在`kernel/proc.c`中添加获取状态不是`UNUSED`的进程数


### 收获

- 了解xv6进程管理和内存管理
- 了解将内核态数据复制到用户态的方法`copyout()`
- **简单实现真的可以非常简单**：
	* pcb可以简单使用固定长度数组，然后遍历查找
	* 内存分配可以简单使用链表以页为单位链接，分配一次分配一页

## Optional challenge exercises

- Print the system call arguments for traced system calls (easy).
- Compute the load average and export it through sysinfo(moderate).

