---
title: xv6 code review
date: datetime
tags: 
- OS
mathjax: true
---

# Abstract


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

