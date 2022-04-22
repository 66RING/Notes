---
title: OS中一些反直觉的tips
author: 66RING
date: 2021-04-21
tags: 
- os
mathjax: true
---

# Abstract


# Preface


# Overview

# fork and print

- fork复制资源
- print内有缓冲区

## case 1

如下程序会打印多少`-`: 

```c
#include<stdio.h>
#include<sys/types.h>
#include<unistd.h>

int main() {
	int i;
	for(i=0;i<2;i++) {
		fork();
		printf("-");
	}
	return 0;
}
```

答案是8，因为print有缓冲区，print没换行情况缓冲区，缓冲区继承到了子进程。

```
            -> "__"
    -> "-" (不打印，在buffer)
            -> "__"
main 
            -> "__"
    -> "-"
            -> "__"
```


## case 2

如果是`print("-\n");`又打印多少个？答案是6个

**PS: vim下存在显示问题** 。如vim下输入命令`:term ./a`执行该程序，会打印不确定个'-'


## case 3

如果`return`换成`_exit(0)`呢？什么都不打印，因为`_exit()`不会调用退出处理函数，不会清空缓冲区。而`exit()`会情况缓冲区和做退出处理。



# 为什么在vfork中子进程先执行

那得需要了解vfork的应用场景

vfork会让子进程先执行，直到子进程退出或`exec`父进程才会执行。那么就有如下场景：

- 父进程需要开子进程完成某些操作后才能继续执行
- 父进程要exec，但是要保证子进程已经执行完成




