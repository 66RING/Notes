---
title: qemu磁盘文件打开过程
author: 66RING
date: 2021-08-15
tags: 
- qemu
mathjax: true
---

# 磁盘文件打开过程

通过gdb追踪openat系统调用得知，raw格式文件会使用`raw_open_common()`函数打开，调用栈如下：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/virtio_rw/openfd.png" alt="">

通过分析源码得到qemu打开磁盘文件的过程：QEMU使用一个BlockDriverState结构体来管理块设备驱动信息，其中就包括了操作和数据。块设备创建的关键就在于初始化BlockDriverState结构体。

```c
struct BlockDriverState {
	...
	BlockDriver *drv;  		// opeartions
	void *opaque;   		// data
	QTAILQ_ENTRY(BlockDriverState) bs_list;
	...
}
```

`drv`成员表示该设备使用的驱动，提供对设备读、写、探测等的操作方法;`opaque`成员表示具体格式的数据，会在具体的操作函数中类型转换成需要的信息。如：指定以raw格式创建磁盘文件，则`drv`成员中将是`raw`相关的操作函数;而`opaque`将执行`raw`格式数据相关的结构体，即`BDRVRawState`。这样有了BlockDriverState结构就能对特定格式的文件(`.opaque`)使用特定的驱动`(.drv)`了。

其中初始化BlockDriverState结构的主函数是`bdrv_open_inherit()`。由于qemu持支多种协议访问文件，如使用nbs协议访问网络块设备(`-drive nbs:file=xxx`)，使用file协议访问本地文件(`-drive file=xxx`)等。所以在`bdrv_open_inherit()`会解析命令行指定的协议类型、解析指定的文件格式等来找到正确的驱动。

```
static BlockDriverState *bdrv_open_inherit() {
	...
	ret = bdrv_fill_options(&options, filename, &flags, &local_err); // 设置一些默认选项
	...
	drvname = qdict_get_try_str(options, "driver"); 			// 如果命令行指定format参数，解析找到驱动
    if (drvname) {
        drv = bdrv_find_format(drvname);
        if (!drv) {
            error_setg(errp, "Unknown driver: '%s'", drvname);
            goto fail;
        }
    }
	...
	bs->probed = !drv; 							// 如果命令行没指定format，则
    if (!drv && file) {
        ret = find_image_format(file, filename, &drv, &local_err);
        if (ret < 0) {
            goto fail;
        }
	}

}
```

`bdrv_open_inherit`中会通过尝试解析命令行字符串的方式找到对应的format和protocol，也会通过读取文件内容做解析的方式查找：

```
static int find_image_format(){
	uint8_t buf[BLOCK_PROBE_BUF_SIZE];
	...
	ret = blk_pread(file, 0, buf, sizeof(buf));
	drv = bdrv_probe_all(buf, ret, filename);
	...
}
```

上面的代码就展示了它读取文件内容做探测的过程。

这里只考虑qemu打开本地文件作为块设备的方法，即使用file协议创建块设备的方法。

`bdrv_open_inherit()`中会调用`bdrv_open_common()`来初始化BlockDriverState的`.opaque`成员。

```c
static int bdrv_open_common(...){
	...
	driver_name = qemu_opt_get(opts, "driver");
    drv = bdrv_find_format(driver_name);
	...
	ret = bdrv_open_driver(bs, drv, node_name, options, open_flags, errp);
	...
}
```

首先解析命令行参数，使用`bdrv_fine_format`找到对应的驱动`drv`，之后在`bdrv_open_driver`中使用驱动`drv`对文件进行读、写、打开等操作。

```
static int bdrv_open_driver(BlockDriverState *bs, BlockDriver *drv,
                            const char *node_name, QDict *options,
                            int open_flags, Error **errp)
{
    ...
    bs->drv = drv;
    bs->read_only = !(bs->open_flags & BDRV_O_RDWR);
    bs->opaque = g_malloc0(drv->instance_size);

    if (drv->bdrv_file_open) {
        assert(!drv->bdrv_needs_filename || bs->filename[0]);
        ret = drv->bdrv_file_open(bs, options, open_flags, &local_err);
    } else if (drv->bdrv_open) {
        ret = drv->bdrv_open(bs, options, open_flags, &local_err);
    } else {
        ret = 0;
    }
	...
}
```

以打开raw格式的文件纹理在`bdrv_open_driver`中调用了`raw_open_common()`回调函数对文件进行打开，对应上面的`drv->bdrv_file_open()`。在`raw_open_common()`中就用了raw格式的打开方法，从而创建BlockDriverState的`opaque`成员。

```c
static int raw_open_common(BlockDriverState *bs, QDict *options,
                           int bdrv_flags, int open_flags,
                           bool device, Error **errp)
{
    BDRVRawState *s = bs->opaque;
	...
	...
	fd = qemu_open(filename, s->open_flags, errp);
	s->fd = fd;
	...
}
```

至此BlockDriverState结构初始化完毕，驱动由`drv`成员管理，设备的数据由`opaque`成员管理。

BlockDriverState中的`opaque`成员是个`void *`的类型，操作时由`drv`指向的驱动转换为正确的类型。如上面说的，raw格式文件用`BDRVRawState`结构存储，同理vmdk类型的文件对应的就是vmdk的driver：`BlockDriver bdrv_vmdk`，再由driver取出需要的类型：

```
vmdk_co_preadv(BlockDriverState *bs, uint64_t offset, uint64_t bytes,
               QEMUIOVector *qiov, int flags)
{
	BDRVVmdkState *s = bs->opaque;
	...
}
```

# 使用uImage启动和我们的启动方式的差异

uImage是专门为uboot设计的一种镜像格式。

```
$ file uImage
uImage: u-boot legacy uImage, Linux-3.16.0, Linux/PowerPC, OS Kernel Image (gzip), 4134148 bytes, Fri Dec 27 09:34:13 2019, Load Address: 0x00000000, Entry Point: 0x00000000, Header CRC: 0x460B37DD, Data CRC: 0x392248B2
```

可以使用uboot的`bootm`命令启动存在内存中的镜像。命令格式为`bootm <img> <ramdisk> <fdt>`

启动`ramdisk`表示ramdisk的位置，`fdt`表示设备树文件的位置，这两个参数是可以省略的，可以传递`-`表示参数为空。


