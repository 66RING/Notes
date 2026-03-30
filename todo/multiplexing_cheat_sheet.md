---
title: 多路复用小抄
author: 66RING
date: 2023-02-26
tags: 
- multiplexing
mathjax: true
---

# select

1. 定义fd集: read fds, write fds, except fds
2. 零初始化fd集合
3. 使用API向set中填充需要监听的fd, 并记录最大fd
4. 监听, 传入最大fd和超时时间
5. 使用API判断fd是否就绪: `FD_ISSET(fd, &set)`
6. 清空fd

```c
int process(int fd) {
    fd_set rfds, wfds, efds;
	int maxfd;
    // 不超时置空, 非阻塞置0
	struct timeval *tvp = time_out();
    FD_ZERO(&rfds);
    FD_ZERO(&wfds);
    FD_ZERO(&efds);
	FD_SET(fd, &rfds);
	FD_SET(fd, &wfds);
	FD_SET(fd, &efds);
	maxfd = fd;
	int ret = select(maxfd + 1, &rfds, &wfds, &efds, tvp);
	if (FD_ISSET(fd, &rfds) || FD_ISSET(fd, &wfds) || FD_ISSET(fd, &efds)) {
		handler();
		FD_CLR(fd, &rfds);
		FD_CLR(fd, &rfds);
		FD_CLR(fd, &wfds);
		FD_CLR(fd, &efds);
	}
}


```

# epoll
