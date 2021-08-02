---
title: Linux中分区扩容与磁盘数据迁移
date: 2021-07-29
tags: 
- linux
- tips
mathjax: true
---

# 分区扩容

**注意，这种方法只能用于最后一个分区的扩容** 

使用`lsblk`查看现有分区情况，如：

```sh
$ lsblk
vda    253:0    0    12G  0 disk
├─vda1 253:1    0     1K  0 part
├─vda2 253:2    0     2G  0 part [SWAP]
└─vda3 253:3    0     8G  0 part /
```

使用fdisk进入要分区编辑界面

```sh
$ fdisk /dev/vda

Command (m for help):
```

删除原分区

```
Command (m for help): d
Partition number (1-3): 3
```

在**相同起始位置新建分区**:

```
Command (m for help): n
```

告诉文件系统修改了大小。ext系列文件系统可以使用`resize2fs`命令，xfs文件系统可以用`xfs_growfs`命令

```
$ resize2fs /dev/vda1
```


# 数据迁移

考虑旧的boot分区容量爆满，我们要将它迁移到一块新的容量更大的磁盘的情况。

接入新磁盘，通过`lsblk`查看新磁盘：

```sh
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda     	259:0    0 476.9G  0 disk
├─sda1 		259:1    0   512M  0 part /boot
├─sda2 		259:2    0 460.4G  0 part /
└─sda3 		259:3    0    16G  0 part [SWAP]
sdb 		 			 128G  0 disk
```

假设我们的新磁盘为`/dev/sdb`，我们要将原`/dev/sda1`boot分区的内容迁移到`/dev/sdb1`。

首先使用`fstab`为新磁盘创建分区。略

然后使用`dd`命令迁移数据。`dd`会迁移每个字节，**包括分区信息以及UUID**

```sh
$ dd if=/dev/sda1 of=/dev/sda status=progress
```

修改文件系统大小

```
resize2fs /dev/sdb1
```

由于`dd`会将uuid一并复制，所有需要修改uuid

```
tune2fs -U random
```

这时我们的boot分区已经迁移到新磁盘，而且使用新的uuid，所有需要修改boot相关的配置，文件系统挂载、引导分区：

- fstab
- grub

修改fstab文件，将boot分区的UUID修改为新的boot分区的UUID

```sh
$ vim /etc/fstab
```

同时grub引导程序中仍然使用就得boot分区，所有也需要进行修改

```sh
grub-install /dev/sdb
```

```
grub-mkconfig -o /path/to/new/bootfile
```



