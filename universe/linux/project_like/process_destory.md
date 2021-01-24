---
title: 进程退出和销毁过程分析
date: 2021-01-15
tags: linux, kernel, project
mathjax: true
---

# 进程退出和销毁过程分析

结束一个进程的生命可以分为两个步骤：进程退出和进程销毁。进程退出主要是释放进程的资源，使进程称为僵死(`TASK_ZOMBIE`)状态;进程销毁主要是通过父进程，释放僵死进程的各种信息

当前进程被设为`TASK_ZOMBIE`僵死状态后会向其父进程发生SIGCHLD信号，父进程收到SIGCHLD信号后父进程会销毁僵死状态的子进程。

父进程通过调用`wait()`或`waitpid()`函数等待SIGCHLD处理僵死进程


## 1 进程退出

进程退出通常的通过系统调用执行`do_exit`函数。简单的说`do_exit`主要执行如下流程：

- 设置和重置标志位
    * 如设置退出码`exit_code`，设置进程状态`TASK_ZOMBIE`，设置退出状态`PF_EXITING`等
- 释放资源
    * 释放内存，信号量，打开的文件的描述符等
- 给子进程找养父
    * 因为进程的销毁依赖于父进程，没有养父的孤儿进程将会造成资源浪费

主要会释放如下资源：

```c
exit_signals(); // sets PF_EXITING
exit_mm();      // 释放内存和描述符 ?我不太明白mm和一个数组的内存的关系
exit_sem();     // 释放信号量
exit_shm();
exit_files();   // 释放打开的描述符
exit_fs();      // 释放文件系统数据
exit_task_namespace();
exit_task_work();
exit_thread();
```

最后通过`exit_notify()`向父进程发出SIGCHLD信号销毁进程，然后执行`do_task_dead()`调用`schedule()`触发调度，进程终止。

下面主要分析进程退出的内存释放过程


### 1.1 退出过程的内存管理

进程的内存释放主要在`exit_mm`中的执行，其执行流程如下：

```c
mm_release
    deactivate_mm  // 删除缓存的寄存器状态
mmput()            // 真正释放内存的函数
    exit_mmap()
        arch_exit_mmap
        free_pgtables
    mm_put_huge_zero_page()
    mmdrop() -> __mmdrop()  // 释放内存描述符和pdg表
        mm_free_pgd() -> pgd_free
        free_mm() -> kmem_cache_free
```


## 2 进程销毁

进程终止后会处于僵死状态，需要父进程执行wait操作来回收进程描述符以及内核栈所占内存，同时把僵死进程从相关的各个表上移除。处理函数为`release_task`

`release_task`主要有如下过程：

- 递减进程数：`atomic_dec(&p->user->processes)`
- ?如果进程被追踪，调用`ptrace_release_task`，把它从调试程序的`ptrace_children`链表中删除
    * 底层调用`__ptrace_unlink`
- `__exit_signal(p)`，删除所有挂起信号并释放`signal_struct`描述符
- `__unhash_process`
- 调用`sched_exit()`函数来调整父进程的时间片

