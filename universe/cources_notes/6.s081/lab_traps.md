---
title: MIT6.S081 traps lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# Traps

## riscv assembly

通过`make fs.img`的可以生成`call.c`对应的汇编代码辅助阅读和学习。如根据需求修改`call.c`然后时候`make fs.img`生成`call.asm`，就可以查看对应汇编内容。

回答以下问题

- 哪些寄存器用来保存函数?
	* `a0~a2`
- 汇编代码如何执行函数调用的?
	* `f`和`g`函数分别在哪调用?
		+ 编译优化成了`li	a1,12`
- 函数`printf`的地址是?
	* `0x628`
- `jalr ra`中ra寄存器的值?
	* `auipc rd, imm` => pc加上立即数，结果保存在rd寄存器中
	* `jal`无条件跳转
	* 计算后`ra`值为`0x30`
- 下面代码的输出? `HE110 World`
	* riscv是小端机
	* `hex(57616) = 0xe110`
	* `0x00646c72`从地址从低到高分别是`rld`
```
	unsigned int i = 0x0064(d)6c(l)72(r);
	printf("H%x Wo%s", 57616, &i);
```
- 下面的代码中`y=?`输出什么?为何发生这种事情?
	* `printf("x=%d y=%d", 3);`
	* 输出不定，因为参数通过`a2`寄存器传入，这里没有指定参数。`a2`寄存器的值可能在某个地方被设置


## backtrace

在`kernel/printf.c`中实现`backtrace()`，打印当前错误的函数调用栈(保存的函数返回地址)。在`sys_sleep`中调用，然后执行`bttest`测试。

**栈结构布局**

```
|-----------------|
| Return address  |
| To prev frame   |
| Saved registers |
| Local variable  |
|-----------------|
| ...             |
| ...             |
|-----------------|
| Return address  | <- fp
| To prev frame   |
| Saved registers |
| Local variable  | <- sp
|-----------------|
```

检测实验结果:
1. 在`sys_sleep`中添加`backtrace()`
2. 运行`bttest`会得到一串地址序列
3. 使用`addr2line`工具查看对应的调用函数
	- `addr2line -e kernel/kernel`启动 
	- 程序是交互式的，将地址粘贴入看返回结果，使用`<c-D>`退出


### hint

- 在`kernel/defs.h`中声明backtrace
- **gcc编译器会保存当前执行函数的frame pointer到s0寄存器**。使用如下函数读取frame pointer
```
// kernel/riscv.h
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```
- [栈帧结构](https://pdos.csail.mit.edu/6.828/2020/lec/l-riscv-slides.pdf)，返回值与frame pointer偏移固定(-8)，保存的frame pointer与frame pointer偏移固定(-16)
- xv6为栈分配一页的空间，地址是页对齐的，所以可以通过`GROUNDDOWN(fp)`和`PGROUNDUP(fp)`计算栈地址和栈结束地址(`kernel/riscv.h`)
- 在`kernel/printf.c:panic()`中调用


### 收获

- 栈中由许多栈帧组成，每个栈帧记录函数调用的信息
- 栈帧的指针与编译器相关，gcc会将当前栈帧指针保存到`s0`寄存器中
- 内敛汇编的使用(读取`s0`寄存器)
- 栈帧结构
- **`addr2line`**工具可以将函数地址翻译成函数名(所在文件和位置)


## alarm

(根据cpu时间)周期alert进程。即使用时钟中断。

实现`sigalarm(n, fn)`在n个ticks(cpu时间)后执行fn。`sigalarm(0, 0)`则停止周期调用

- 添加`alarmtest`到makefile
- 添加syscall：`sigalarm`和`sigreturn`。让编译通过
- `make fs.img`后可以通过对应`asm`查看汇编代码，方便调试
- 测试`alarmtest`和`usertests`


### test0 定期执行handler

- 添加syscall，包括`user.h`, `sysproc.c`, `syscall.c`等。直到编译成功
- 实现`sigalarm`函数主体，记录间隔和函数指针，修改`struct proc(proc.h)`用于保存
- 记录"流逝的时间"，因为是**CPU**时间，同样需要修改`struct proc`
- 基于上述修改在`allocproc()`中初始化proc
- 每个tick都会触发时钟中断。即修改中断入口`kernel/trap.c:usertrap()`，当时钟到期执行回调函数
	* 时钟中断在`if(which_dev == 2)`的判断中
	* 注意`sigalarm(0, 0)`，interval和fn**同时**为0的情况。而fn是可能为0的。`alarmtest.asm`
- **当trap返回用户空间(计时器到期)：返回地址怎么确定的?**
	* `trapframe`，见`usertrapret()`
	* **用户空间回调函数不能直接在内核空间调用**，要返回用户空间调用，然后再`sigreturn`回到内空间
- 调试`make CPU=1 qemu-gdb`
- 直到打印`alarm!`才算成功

返回当然还存在问题，test0先把到期调用实现了。运行`alarmtest`


### test1/test2 恢复中断的代码

当handler执行完成返回用户空间

- 保存/恢复寄存器 => 需要保存哪些寄存器?
- 修改`usertrap()`以便可以返回到中断的用户代码
- 完成`sys_sigreturn()`主体，与`sys_sigalarm`相互配合
- 防止重入未返回的handler(test2)

这里的实现是简单地将`trapframe`备份，然后恢复即可。


## 收获

- trapframe和context都有"保存现场"，有何区别?
	* context主要负责进程间的现场切换。trapframe负责用户空间和内核空间的现场切换
- 中断入口和基本处理方法
- 初见trapframe


## Optional challenge exercises

Print the names of the functions and line numbers in backtrace() instead of numerical addresses (hard).

TODO



