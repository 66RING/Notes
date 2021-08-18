---
title: qemu如何由事件循环找到真正要操作的目标设备
date: 2021-08-14
tags: 
- qemu
- virtio
mathjax: true
---

# 从eventfd联系到fd的流程

非阻塞事件循环中，通过修改eventfd来通知qemu有事情需要qemu处理，如调用读写回调函数等。已知eventfd仅做通知的作用，qemu事件循环能够监听到eventfd的改变然后做相应处理。但是qemu如何由eventfd找到真正要进行处理的对象呢(如由磁盘读写关联的eventfd找到磁盘的fd进行读写)？本文将以virtio的磁盘读写为例，介绍从eventfd找到fd的过程。


## 研究方法

思路：qemu使用主机的一个文件作为虚拟机的磁盘(以下成该文件为磁盘文件)，则必须会使用打开文件的系统调用，而qemu会通过打开文件的fd进行读写操作。因此使用gdb的调试功能先追踪到文件的fd，在访问fd时触发断点，打印调用栈，再根据调用栈信息分析源码即可得知eventfd到fd的流程。

使用strace观察qemu打开磁盘文件使用的系统调用：

```
$ strace qemu-system-riscv64 -M virt \
	-m 2048M \
	-kernel ./Image \
	-drive file=rootfs.img,format=raw,id=hd0,if=virtio  \
	-append "root=/dev/vda rw console=ttyS0" \
	2>&1 | grep -i rootfs

execve("/usr/local/bin/qemu-system-riscv64", ["qemu-system-riscv64", "-M", "virt", "-m", "2048M", "-kernel", "./Image", "-drive", "file=rootfs.img,format=raw,id=hd"..., "-append", "root=/dev/vda rw console=ttyS0"], 0x7fff4b54e5a0 /* 89 vars */) = 0
openat(AT_FDCWD, "rootfs.img", O_RDONLY|O_NONBLOCK|O_CLOEXEC) = 12
newfstatat(AT_FDCWD, "rootfs.img", {st_mode=S_IFREG|0644, st_size=2147483648,
...}, 0) = 0
...
```

可见磁盘文件通过`openat`系统调用打开。启动一个虚拟机，通过qemu monitor添加一个磁盘文件，在gdb中监听`openat`系统调用，最后可以发现qemu对raw格式的文件使用`raw_open_common`做打开操作。

```
(gdb) catch syscall openat
```

这里使用的是一个riscv虚拟机。以raw格式打开一个磁盘文件为例，已知raw格式文件用`raw_open_common`函数打开。

```sh
qemu-system-riscv64 -M virt \
  -m 2048M \
  -kernel ./Image \
  -drive file=rootfs.img,format=raw,id=hd0,if=virtio  \
  -append "root=/dev/vda rw console=ttyS0" \
```

```
static int raw_open_common(BlockDriverState *bs,...)
{
    BDRVRawState *s = bs->opaque;
	...
	fd = qemu_open(...);
	s->fd = fd;
	...
}
```

使用gdb的`watch`命令监听对`s->fd`改变:

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/watch_fd.png" alt="">


**注意** 因为这里riscv虚拟机中途会reopen磁盘镜像导致fd改变，所以在监听到fd改变后重新定位并用`rwatch`监听对新fd访问事件，如下：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/reopen.png" alt="">

待虚拟机启动稳定后观察fd的读写，打印调用栈：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/bt_snipt.png" alt="">

可以发现读写在协程中进行：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/bt_comp.png" alt="">

gdb中使用`finish`从协程返回，观察入口以及调用情况：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/handler.png" alt="">

通过分析调用流程源代码得到如下调用流程：


## eventfd到fd的流程

qemu事件循环监听到eventfd改变后进入dispatch阶段。dispatch阶段主要通过`aio_dispatch_handler()`完成，根据类型执行read/write回调：

```c
static bool aio_dispatch_handler(AioContext *ctx, AioHandler *node)
{
	...
	if(... && node->io_read)
		node->io_read(node->opaque);

	if(... && node->io_write)
		node->io_write(node->opaque);
	...
}
```

这里使用的是virtio，会使用`virtio_queue_host_notifier_aio_read(EventNotifier node->opaque)`回调函数

```
static void virtio_queue_host_notifier_aio_read(EventNotifier *n)
{
    VirtQueue *vq = container_of(n, VirtQueue, host_notifier);
    if (event_notifier_test_and_clear(n)) {
        virtio_queue_notify_aio_vq(vq);
    }
}
```

将eventfd(EventNotifier)和fd联系起来的 **核心是`container_of()`宏** ，这个宏从EventNotifier获取到VirtQueue，而VirtQueue中就保存着Virtio设备相关的数据，其中就包括打开的磁盘文件的fd，从而能够由eventfd找到fd。

```
struct VirtQueue
{
	...
	VirtIODevice *vdev;
    EventNotifier guest_notifier;
    EventNotifier host_notifier;
	...
}
```

**因此可以大胆猜测一下：qemu就是使用container_of()宏由eventfd(EventNotifier)联系到具体设备的**。而具体设备中就保存了设备的各种信息，从而完成对具体设备的操纵。

之后vq中的回调函数完成读写。这里virtio是以启动协程的方式来做读写的，调用如下：

```
virtio_queue_notify_aio_vq(VirtQueue *vq) 调用--> vq->handle_aio_output(vdev, vq);
	virtio_blk_data_plane_handle_output(VirtIODevice *vdev, VirtQueue *vq)
		VirtIOBlock *s = (VirtIOBlock *)vdev;
		virtio_blk_handle_vq(VirtIOBlock *s, VirtQueue *vq)
			virtio_blk_submit_multireq(s->blk, &mrb);
				submit_requests
```

`submit_requests()`中判断是读还是写操作，然后启动相应的读/写协程。以读操作为例由`blk_aio_preadv()`调用`qemu_coroutine_create(CoroutineEntry *entry, void *opaque)`启动协程。

`blk_aio_preadv()`中会将需要的参数打包成`BlkAioEmAIOCB *acb`结构体作为`qemu_coroutine_create()`的第二个参数，而第一个参数entry是`blk_aio_read_entry`函数指针。

所以最终协程启动为：`entry(opaque)`即`blk_aio_read_entry(acb)`。

再分析协程的调用栈，即可解读出设备(VirtIOBlock)到具体fd的过程：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/bt_comp.png" alt="">


## 磁盘文件打开过程

```
-hda

-drive

```

TODO

- 观察完整过程

