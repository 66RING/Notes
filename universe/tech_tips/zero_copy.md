---
title: 零拷贝
author: 66RING
date: 2022-03-30
tags: 
- os
mathjax: true
---

# Preface

用户态拿到硬件数据再发送需要几步？

- 首先硬件是归操作系统管理的，所以要想获取到硬件数据得先进入内核态
- 其次内核态和用户态的地址空间是隔离的，所以需要内核态和用户态直接的数据拷贝

ok，理解了前置基础，let jump into it

# 最原始的操作

理解了上面提到的基础只是那么应该能够想到。以网卡为例

1. 用户读数据 
	- 因为网卡(硬件)归操作系统管理，网卡需要先将数据发给内核buffer
		* 第一次拷贝(DMA)
	- 内核拿到数据后再将buffer的内容拷贝到用户态
		* 第二次拷贝(kernel -> user)
2. 用户处理完后向网卡发送(写)
	- 用户空间的buffer(数据)拷贝到内核的socket buffer
		* 第三次拷贝(user -> kernel)
	- 内核最后将buffer中的数据传输到设备
		* 第四次拷贝(DMA)

可以看到需要四次拷贝操作，四次状态切换(user -> kernel x2, kernel -> user x2)

# mmap

如果我将接收数据的那个内核buffer映射到用户空间，那就不用用户态和内核态之间的拷贝了。

最后只需要"读buffer"到"写buffer"(socket缓冲区)的拷贝和DMA拷贝


# sendfile

假设我们有个场景：读取磁盘文件，然后直接发送给网卡。

那么我们可以用一个智能的api: `sendfile`告诉内核向网卡发送文件，那么内核知道所有所需信息后就可以**全程在内核态执行**

此时需要一次cpu拷贝("读buffer"到"写buffer")和两次DMA拷贝。那么我们可不可以直接从"读buffer"写到设备呢？yes we can


# SG-DMA

如果网卡硬件支持`SG-DMA`那么我们就可以指定直接从"读buffer"写到设备了。that's it 这就是**零拷贝**了

可以看到两次设备的DMA是不可避免的，**最终有两次上下文切换两次DMA拷贝**


# References

- [原来 8 张图，就可以搞懂「零拷贝」了](https://juejin.cn/post/6995519558475841550)
- [一文彻底弄懂零拷贝原理](https://www.cnblogs.com/xiaolincoding/p/13719610.html)
