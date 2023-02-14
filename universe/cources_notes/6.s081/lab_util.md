---
title: MIT6.S081 utils lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---

# lab1

## sleep

编写一个sleep用户程序，可以参考grep(`grep.c`)。

目标:

- 熟悉lab架构：如何向xv6中插入自己编写的程序
- 熟悉一个用户程序的编写流程
- 熟悉编译系统之`UPROGS`
- 熟悉xv6提供的库函数`ulib.c`

solution:

- `user/sleep.c`


## pingpong

编写一个pingpong用户程序：创建一对父子进程, 一个进程使用一读一写两个管道传递数据。父进程向子进程发送数据, children打印`<pid>: received ping`, 然后子进程向父进程发送数据，然后子进程退出。父进程接收，然后打印`<pid>: received pong`, 之后退出。

目标:

- 熟悉`pipe`, `fork`, `read`, `write`等用法


## primes

使用管道设计一个并发版的质数查找器: [算法](https://swtch.com/~rsc/thread/)。

目标:

- 进一步熟悉`pipe`和`fork`的使用
- 了解文件描述符的控制
- **只有当管道的两个方向`fd[2]`都关闭时才算关闭，子进程`read`才会返回0**

错误：

```c
if(pid == 0) {
	while(read(fd[0], &num, 4)!=0) {}
} else {
	write(fd[1], &num, 4);
	close(fd[1]);
	close(fd[2]);
	int stat;
	wait(&stat);
}
```

正确：

```c
if(pid == 0) {
	close(fd[1]);
	while(read(fd[0], &num, 4)!=0) {}
} else {
	write(fd[1], &num, 4);
	close(fd[1]);
	close(fd[2]);
	int stat;
	wait(&stat);
}
```

## find

写一个find程序

目标:

- 了解文件系统，至少目录/文件是怎么表示的`struct dirent`, `struct stat`
- 了解文件系统如何读取的`fstat`, `read(fd, &de, sizeof(de)) == sizeof(de)`


TIPS:

- 记得关闭文件描述符


## xargs

实现一个xargs程序，xargs能将标准输入变为程序的命令行参数。如`echo "1\n2" | xargs -n 1 echo line`输出：

```
line 1
line 2
```

目标:

- 搞清楚程序在操作系统中是如何产生，如何使用管道交互的
- 搞清楚`exec`和`fork`的流程


## Summary

## Optional challenge exercises

TODO

