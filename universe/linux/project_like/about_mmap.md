---
title: execv中地址地址映射流程分析
date: 2021-01-18
tags: 
- linux
- kernel
- project
mathjax: true
---

## 1 execv函数地址映射流程分析

### 1.1 execv如何填充新进程的页表

execv()中会调用`bprm_mm_init`函数准备一个`linux_binprm`结构体，该结构体用于描述新进程的各种信息，最后由`search_binary_handler`找到对应文件格式的处理程序加载执行`load_binary`。

其中`bprm_mm_init`会执行如下流程：

```
bprm_mm_init()
    bprm->mm = mm_alloc()
        allocate_mm -> kmem_cache_alloc(mm_cachep, GFP_KERNEL) // 分配内存
        mm_init  // 初始化设置
            mm_pgtables_init(mm) // 置0
            mm->pdg = mm_alloc_pgd -> pgd_alloc // 申请pgd
            init_new_context()   // ???
    __bprm_mm_init()   // 初始化bprm：设置bprm->vma
        vm_area_alloc
            kmem_cache_zalloc (slab?)
            vma_init
        arch_bprm_mm_init
```

**什么时候填充的elf格式到页表?** 虚页都映射到了正确的物理页

好像pdg探测到都是0，所有应该的demand allocation?但不知道是用的哪个函数


### 1.2 execv中内存映射

内存映射的定义：将磁盘文件系统的文件的某一部分或者块设备文件映射到一段连续的内存，这样一来在内存上的改动就对应的转换成了对文件相应位置的改动。

?? 文件数据的存储通常不是连续的，内核利用`address_space`结构，提供一组方法从backing store读取数据，将映射的数据表示成连续的线性取，提供给内存管理子系统。


#### 1.2.1 vm_area_struct

linux使用`vm_area_struct`结构体来描述一段连续的、具有相同访问属性的虚存空间。每个segment用一个`vm_area_struct`结构体表示。

新建的vma相邻的vma有相同的属性，且基于相同的映射对象（比如是同一个文件），则还会产生vma的合并。减少vma的数量有利于减轻内核的管理工作量，降低系统开销。如果没有发生合并，则需要调用`insert_vm_struct`在vma链表和vma红黑树中分别插入代表新vma的节点。

一个进程是所有线性区是通过链表或树形结构连接起来的。`vm_area_struct`结构体中包含区域的起始和终止以及其他相关信息，也包含一个`vm_ops`指针用于引出所有可以对这个区域进程操作的函数。

mmap函数的作用就是创建一个新的`vm_area_struct`，并将其与文件的物理存储联系起来。


#### 1.2.2 address_space

`address_space`用来管理文件(`struct inode`)映射到内存页面(`struct page`)。对应的`address_space_operations`就是用来操作该文件映射的操作。

一个具体的文件在打开后，内核会在内存中为之建立一个`struct inode`结构，其中的`i_mapping`域指向一个`address_space`结构，这样，一个文件就对应一个`address_space`结构。


#### 1.2.3 execv中内存映射的流程

execv会执行`do_open_execat`打开文件，并填充相应file结构体，其中包括`file_operations`。然后`search_binary_handler`寻找对应格式的处理程序。

对于elf格式的文件，对应的处理程序一般是`load_elf_binary`，所以`load_binary`函数指针指向`load_elf_binary`。在`load_elf_binary`中会执行`elf_map`建立与可执行文件的映射，`elf_map`执行流程如下：

```c
elf_map
    vm_mmap->vm_mmap_pgoff
        do_mmap_pgoff
            do_mmap
                get_unmapped_area  // obtain addr to map to  
            mmap_region
                vm_area_alloc
                    kmem_cache_zalloc   // 分配vma空间
                    vma_init(vma, mm)   // 初始化vma
                        vma->mm = mm;
                        vma->vm_ops =  &vm_operations_struct {};
                call_mmap(file, vma)  // 调用file->op->mmap进行内存映射
```

主要做了这么几个事情：

- 1 寻找一段空闲的满足要求的连续虚拟地址
- 2 为这个虚拟区分配一个`vm_area_struct`结构
- 3 初始化vma，插入vma链表或树中
- 4 通过待映射的文件`struct file`找到对应的`file_operations`，调用`mmap(file, vma)`(在`call_mmap`这个宏中)建立映射

不同的环境打开同一文件时会生成不同的`struct file`实例，如使用的是xfs文件系统，在生成file实例时`file_operations`用的就是`xfs_file_operations`类。其中会找到对应`address_space`中相应的read，write，mmap等方法。

虚拟区域建立完成并完成了地址映射，但是没有将文件数据拷贝到主存。这是出于效率考虑的demand paging(按需调页)机制。当发现一段地址不存在物理页面时触发缺页异常，执行对应的调页方法(`vma->vm_ops->fault`)。

该部分处理具体的方法依赖于映射到发生异常的地址空间（`address_space`中的文件，因此需要调用特定于文件的方法来获取数据。如xfs中该方法指向`vm->vm_ops->fault=xfs_filemap_fault`

再由`xfs_filemap_fault`执行对应文件的`filemap_fault`，其中`xfs_filemap_fault`，主要还是执行通用的`filemap_fault`方法，主要流程如下：

```c
xfs_filemap_fault
    filemap_fault     // xfs_filemap_fault中主要还是执行通用的filemap_fault
        find_get_page // 申请page，首先会看看有没有缓存
        // 如果没有缓存就先申请缓存
            __page_cache_alloc
        vmf->vma->vm_file->f_mapping->a_ops->readpage()  // 使用文件对应的read方法执行映射操作
            xfs_vm_readpage  // readpage指针指向xfs_vm_readpage
                mpage_readpage   // xfs_vm_readpage底层调用mpage_readpage
```



