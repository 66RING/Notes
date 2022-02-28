---
title: 如何让已经运行的程序到后台运行
author: 66RING
date: 2022-02-27
tags: 
- tech
mathjax: true
---

1. `<c-z>`挂起, 再开tmux, 在tmux里`fg`
	- `jobs`
	- `fg`
	- `bg`
2. `disown <pid>`shell退出不影响程序
3. 中途改变想法了想记录日志，可以使用调试器修改描述符，从而记录日志
```
$ gdb -p <pid>
(gdb) P (int)dup2((long)open("path/to/file", 123, 123), 1)  // 改标准输出到日志文件
(gdb) detach
```
4. 更彻底的方法`reptyr`

