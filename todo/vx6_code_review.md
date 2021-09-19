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

`file[:mainfunction][, descreption]`

```
<function>[: desc]

[main flow]

[sub function]
[- sub function desc]

...
```

# Overview

## kernel address

### kernel address mapping(kernel page table)

`kernel/vm.c:kvminit`

- `kernel_pagetable`, kernel page table pointer

```
kvminit: init kernel page table

kvminit -> kvmmake -> kvmmap -> mappages -> walk// mapping address

walk:
- find page tabe/pte
- insert pa
```

### write pt to page table register

`kernel/vm.c:kvminithart`

```
kvminithart

sfence_vma:
- flush TLB, because page table change
```

## physical memory

### init allocator

`kernel/kalloc.c`, free list allocation

```
kinit 

kinit -> freerange -> free

freerange
- free a range of page into free list
```


### heap(sbrk)

`kernel/proc.c`

```
growproc, alloc and assign new page size

growproc
- myproc, get current process page table
- positive n, call uvmalloc to alloate
- nagetive n, call uvdemalloc to dealloate
```

### exec


# Infrastructure

## syscall[TODO]

- trapframe

`kernel/syscal.c`

```
syscall
```

