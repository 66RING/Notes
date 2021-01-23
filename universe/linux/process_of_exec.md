---
title: execv的执行过程
date: 2021-01-14
tags: linux, kernel, exec
mathjax: true
---

# execv执行过程

## 可执行程序的表示

### elf文件格式

Linux下标准的可执行文件格式是ELF(Executable and Linking Format)，是一种对象文件的格式，用于定义不同类型的对象文件(Object files)中都放了什么东西、以及都以什么样的格式去放这些东西。

但是linux也支持其他不同的可执行程序格式, 各个可执行程序的执行方式不尽相同, 因此linux内核每种被注册的可执行程序格式都用`linux_binfmt`来存储, 其中记录了可执行程序的加载和执行函数

同时我们需要一种方法来保存可执行程序的信息, 比如可执行文件的路径, 运行的参数和环境变量等信息，即`linux_binprm`结构


### linux_binprm结构体

使用`linux_binprm`结构来保存可执行程序的信息，如可执行程序路径、运行的参数和环境变量等。


### linux_binfmt结构体

linux内核中，每种被注册的可执行程序都用`linux_binfmt`来存储，用于记录可执行程序的加载和执行函数。

linux支持其他不同格式的可执行程序，即linux能运行其他操作系统所编译的程序，因此linux内核使用`linux_binfmt`来描述各种可执行程序。其提供了3种方法来加载和执行可执行程序：

- `load_binary`
    * 通过读存放在可执行文件中的信息为当前进程建立一个新的执行环境
- `load_shlib`
    * 用于动态的把一个共享库捆绑到一个已经在运行的进程, 这是由uselib()系统调用激活的
- `core_dump`
    * 在名为core的文件中, 存放当前进程的执行上下文. 这个文件通常是在进程接收到一个缺省操作为”dump”的信号时被创建的, 其格式取决于被执行程序的可执行类型

当执行一个可执行程序时，内核会`list_for_each_entry`遍历所有注册的`linux_binfmt`对象，对其调用`load_binary`方法来尝试加载，直到加载成功为止


## execv执行过程

执行`execv()`或`execve()`系统调用时，实际的调用内核的`do_execve()`函数。这个函数首先会打开目标映像文件，并从目标文件头部读入若干字节用于填充`linux_binprm`结构体。然后调用`search_binary_handler()`搜索linux支持的可执行文件类型队列，让各种可执行程序来认领和处理改类型的程序。如果类型匹配，则调用`load_binary`函数指针所指向的处理函数来处理目标映像文件。对于elf文件格式`load_binary`处理函数为`load_elf_binary`，故过程为:

```
sys_execve() > do_execve() > do_execveat_common > search_binary_handler() > load_elf_binary()
```


### do_execveat_common执行

`do_execveat_common`依次执行以下操作：

- 调用`unshare_files()`为进程复制一份文件表
- 调用`kzalloc()`分配一份`struct linux_binprm`结构体
- 调用`open_exec()`查找并打开二进制文件
- 调用`sched_exec()`找到最小负载的CPU，用来执行该二进制文件
- 根据获取的信息，填充`struct linux_binprm`结构体中的file、filename、interp成员
- 调用`bprm_mm_init()`创建进程的内存地址空间，为新程序初始化内存管理.并调用`init_new_context()`检查当前进程是否使用自定义的局部描述符表；如果是，那么分配和准备一个新的LDT
- 填充`struct linux_binprm`结构体中的argc、envc成员
- 调用`prepare_binprm()`检查该二进制文件的可执行权限；最后，`kernel_read()`读取二进制文件的头128字节（这些字节用于识别二进制文件的格式及其他信息，后续会使用到）
- 调用`copy_strings_kernel()`从内核空间获取二进制文件的路径名称
- 调用`copy_string()`从用户空间拷贝环境变量和命令行参数
- 至此，二进制文件已经被打开，`struct linux_binprm`结构体中也记录了重要信息, 内核开始调用`exec_binprm`执行可执行程序
- 释放`linux_binprm`数据结构，返回从该文件可执行格式的`load_binary`中获得的代码


### exec_binprm识别并加载二进制程序

每种格式的二进制文加对应一个`linux_binprm`结构体，`load_binary`成员负责识别该二进制文件的格式。

使用`search_binary_handler`函数对`linux_binfmt`结构体链表(链表头为`formats`)扫描，并尝试每个`load_binary`函数，如果成功加载则对`formats`的扫描终止


## Reference

- [http://blog.chinaunix.net/uid-12127321-id-2957869.html](http://blog.chinaunix.net/uid-12127321-id-2957869.html)



