---
title: xv6 code review
date: 2022-05-08
tags: 
- OS
mathjax: true
---

TODO: 整个重启

# Abstract

TODO: start.c

```c
consoleinit();
kinit();         // physical page allocator
kvminit();       // create kernel page table
kvminithart();   // turn on paging
procinit();      // process table
trapinit();      // trap vectors
trapinithart();  // install kernel trap vector
plicinit();      // set up interrupt controller
plicinithart();  // ask PLIC for device interrupts
binit();         // buffer cache
iinit();         // inode table
fileinit();      // file table
virtio_disk_init(); // emulated hard disk
userinit();      // first user process


scheduler();
```

1. `consoleinit()`
	- 初始化uart，配置中断，波特率等
2. `kinit()`
	- 创建堆空间，根据链接脚本中的标号获得堆空间的范围，逐一以页为单位插入到freelist中
3. `kvminit()`
	- 创建内核页表，全局变量`kernel_pagetable`
	- 调用`kvmmake`初始化内核空间，将映射设备地址
		* 使用`KSTACK`为每个进程分配内核栈(映射可读可写)，进程内核栈间间隔一页做guard
4. `kvminithart()`
	- 开启分页: **设置satp寄存器** ，使用SV39的模式
	- 清空TLB
5. `procinit()`
	- 将3中创建好的内核栈分配到各个进程的PCB中，xv6使用定长数组维护pcb
6. `trapinit()`
	- 初始化计时器锁
7. **`trapinithart()`**
	- 内核态中trap
	- 设置Trap-Vector(中断处理函数)：将`stvec`寄存器置为`kernelvec`地址
	- `kernelvec`利用内核栈保存中断现场(push)然后调用`trap.c:kerneltrap()`执行中断服务程序，然后恢复线程(pop)
	- `kerneltrap()`仅做了计时器的处理，让出cpu
8. `plicinit()`
	- Platform Level Interrupt Controller (PLIC)
	- 设置中断优先级，0表示关闭
9. `plicinithart()`
10. `binit()`
	- 初始化buffer cache
	- 设计非常精简，静态数组做空间分配，然后在`binit()`中将数组组织成链表
11. `iinit()`
	- 初始化锁
12. `fileinit()`
	- 初始化锁
13. `virtio_disk_init()`
	- TODO
14. `userinit()`
	- 创建一个初始进程，该进程会`exec("/init")`
	- **`allocproc`**
		* 分配一个PCB
		* **初始化上下文(context)** : 将返回地址设为`forkret`
	- 设置trapframe
		* 设置pc，sp当返回用户态( **`usertrapret`** )时会根据trapframe的内容
15. `scheduler()`
	- PCB数组中查找一个`RUNNABLE`的进程执行
	- 第一次`swtch`将会进入14中设置的`forkret`
	- `forkret`将该进程内核栈的内容保存到trapframe(下次trap就能知道现场)
		* 置`SPP`0, Previous mode, 进入异常前处理器所处模式(S/U)
			+ BTW，MPP可以是M/S/U/
		* 全局中断使能`SIE`
		* 加载用户态页表
		* TODO: `trampoline`


TODO: **内核态用户态切换的机制, trampoline等**


## Trap机制

难中难，重中重

> When it needs to force a trap, the RISC-V hardware does the following for all trap types (other than timer interrupts):
> 1. If the trap is a device interrupt, and the sstatus SIE bit is clear, don’t do any of the following.
> 2. Disable interrupts by clearing SIE. 
> 3. Copy the pc to sepc. 
> 4. Save the current mode (user or supervisor) in the SPP bit in sstatus. 
> 5. Set scause to reflect the interrupt’s cause. 
> 6. Set the mode to supervisor. 
> 7. Copy stvec to the pc.
> 8. Start executing at the new pc.

- ps
- `w_stvec(TRAMPOLINE + (uservec - trampoline));`
	* `readelf -s ./kernel | grep -E "uservec|trampoline"`
	* 发现`uservec`就是`trampoline`, TODO：但是为什么要`uservec - trampoline`呢?

为何trampoline

> Because the RISC-V hardware doesn’t switch page tables during a trap, the user page table must
> include a mapping for uservec, the trap vector instructions that stvec points to. uservec
> must switch satp to point to the kernel page table; in order to continue executing instructions
> after the switch, uservec must be mapped at the same address in the kernel page table as in the
> user page table.


- 内核态trap: 中断和异常
- 用户态trap: 中断，异常和系统调用

- [cool](https://www.cnblogs.com/KatyuMarisaBlog/p/13934537.html)
- [good](https://blog.csdn.net/RedemptionC/article/details/108718347)


```c
uint64 fn = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
// a0: TRAPFRAME, in user page table.
// a1: user page table, for satp.
```


## Deep dive into Trap

难点从用户太trap时，硬件并不会自动切换页表，即satp仍然是用户页表，而且栈顶指针sp可能存在恶意代码

? user -> kernel 不能改pc么，kernel的pc能保存在哪里?

切换地址空间和栈空间

为什么要相同？

- 先切换pc后还能找到切换页表吗?
- 先切换页表(地址空间)后，pc的顺序执行还能正确吗?
- 所以trampoline映射相同内容到相同虚拟地址



# Preface

- satp, page table register

- flow
	* build kernel pt
	* swap kernel pt

layout:

`file[:mainfunction()][, descreption]`

```
<function()>[: desc]

[main flow]

[sub function]
[- sub function desc]

...
```

# Overview

## kernel address

### kernel address mapping(kernel page table)

`kernel/vm.c:kvminit()`

- `kernel_pagetable`, kernel page table pointer

```
kvminit(): init kernel page table

kvminit -> kvmmake -> kvmmap -> mappages -> walk// mapping address

walk:
- find page tabe/pte
- insert pa
```

### write pt to page table register

`kernel/vm.c:kvminithart()`

```
kvminithart()

sfence_vma:
- flush TLB, because page table change
```

## physical memory

### init allocator

`kernel/kalloc.c`, free list allocation

```
kinit()

kinit -> freerange -> free

freerange()
- free a range of page into free list
```


### heap(sbrk)

`kernel/proc.c`

```
growproc(), alloc and assign new page size

growproc()
- myproc, get current process page table
- positive n, call uvmalloc to alloate
- nagetive n, call uvdemalloc to dealloate
```

### exec


# Infrastructure

## TODO

- save to trapframe
- argraw fetch from trapframe
- user mode string to kernel mode
	* fetchstr

## syscall[TODO]


- trapframe

`kernel/syscal.c`

```
syscall()
```

## context switching

`swtch.S`, save old and load new

```
swtch(), save old and load from new

swtch(a0, a1)
- save a0, load a1
- aX = struct context 
```

## sleep and wakeup

### sleep and wakeup

`proc.c:sleep`, explicit sleep to avoid overhead of spinlock

```
sleep()
- p->state = SLEEPING
- p->chan = chan, which means "add to sleep queue". wakeup() just
  traverse all process and check `chan` and `state`
- sched()
```

**tricky things**: lost wakeup.

Because process to sleep is not atomic, it is possible that while p1
call `P()` to `sleep()` and `V()` calling `wakeup()` but wakeup nothing


```
wakeup()
- traverse all proc[], check `chan` and `state`
- change state to RUNNABLE
- wait to schedule
```


### pipe

`pipe.c`: open two file, which `TYPE=FD_PIPE`, as pipe's read/wirte. `struct pipe` maintain the data of pipes.

```
struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read
  uint nwrite;    // number of bytes written
  int readopen;   // read fd is still open
  int writeopen;  // write fd is still open
};

nread/nwirte to manipulate the data[] circulation list
```

```
pipealloc()

pipealloc()
- f = filealloc() to open to file as read/write port
- f->pipe = struct pipe
- f->type = FD_PIPE
```

todo pipeclose

```
pipewrite()

pipewrite()
- base check: open, readable, writeable, etc
- if full then block(sleep) writeble and wakeup who is waiting to read
- else fill buffer until transaction done
```

```
piperead()
- base check
- block if empty
- wakeup write process while read done
```


### wait, exit and kill

`proc.c:exit()`

```
exit(): Exit the current process.  Does not return.

exit()
- close all open file
- drop reference to an in-memory inode
- `reparent()` all children
- wakeup parent
- state to ZOMBIE, set exit state `xstate`
- sched()

reparent()
- traverse all `proc[]`
- set all children's (`if pp->parent == p`) parent to initproc (`pp->parent = initproc`)
```

`proc.c:wait()`

```
wait(): wait child process to exit

wait()
- simply traverse all process, check `if np->parent == p`
- check ZOMBIE `if np->state == ZOMBIE`, found
	* `freeproc()`
- if not found, `sleep()`
```

`proc.c:kill()`: kill process with given pid

```
kill()
- traverse all proc[] to find process with given pid
- set killed
- done, exit until call `usertrap()` to return user space
```

## file system

### buffer cache

# Tricky things

- lost wakeup

