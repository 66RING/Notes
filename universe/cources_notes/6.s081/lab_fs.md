---
title: MIT6.S081 fs lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# File system

添加对大文件和符号链接的支持。可查阅参考书Chapter 8


## Large files

修改xv6支持的最大文件(目前是268block(即BSIZE，BSIZE=1024byte))。

之所以由这样的限制是因为xv6的inode用的12个直接块和1个"singly-indirect"块。所以总的有12+256 = 268block

> 每个块号是uint(8B)，那么一个block(1024B)就可以所以256块

bigfile程序会试图创建最大的文件然后打印大小。如果能够创建大小为65803block的文件则测试通过。

修改xv6的文件系统以使一个inode支持一个"doubly-indirect"。

- 一个"singly-indirect"块能索引256个块
- 一个"doubly-indirect"块能索引256x256个块
- 最终将可以`256*256+256+11 = 65803`
	* 12->11是因为一个用来"doubly-indirect"了

`mkfs`创建的xv6系统镜像文件决定文件系统中块的总数。由`kernel/param.h:FFSIZE`决定。`make mkfs/mkfs`会打印meta块和数据块的详细信息。e.g.

```
$ mkfs/mkfs fs.img
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

PS: 需要重建文件系统的话就`make clean`和rebuild

即，实现一个多重索引方式的


### 如何开始

磁盘inode结构定义在`fs.h:struct dinode`，需要关注`NDIRECT`, `NINDIRECT`, `MAXFILE`和`ADDRS[]`。可以阅读参考书的图8.3。

通过`fs.c:bmap()`来查找磁盘上的文件，请搞清楚`bmap()`的功能。读写文件都会调用`bmap()`，写时`bmap()`会分配新块和间接块来保存文件。

`bmap()`处理两种"块号"。`bn`是"逻辑块号"(即用于文件内部的，相对于文件开头的数)。还要一个块号是用在`ip->addrs[]`和`bread()`的，是磁盘块号。你可以认为**`bmap()`能将文件的逻辑块号映射成磁盘块号**

所以你需要修改`bmap()`以实现`doubly-indirect`块(11个直接块， 1个"singly-indirect"块，1个"doubly-indirect"块)。你不可以修改磁盘上inode的大小。`ip->addrs`的前11个元素表示直接块，12th表示"singly-indirect"，13th表示"doubly-indirect"，这样最终将支持65803块。

测试: `bigfiles`(至少一分半钟哦)


### hints

- 请先理解`bmap()`，写出`ip->addrs[]`的关系图
```
1 block = 1024byte
每项8B(32bit) 1024/8=256项 (一级索引能够索引的块)
两级索引将能够索引256x256项
| [0]direct    | -> | data block(1024byte) |
| [1]direct    | -> | data block           |
| ...          |
| [10]direct   | -> | data block           |
| [11]indirect | -> | direct(256entry)   | -> | data               |
| [12]indirect | -> | indirect(256entry)   | -> | direct(256entry) | -> | data |
```
- 请思考要如何用逻辑块号索引doubly-indirect块
- 如果你修改了`NDIRECT`你可能会需要修改`addrs[]`以保证`file.h:struct inode`和`fs.h:struct dinode`有相同的大小的`addr[]`
- 如果你修改了`NDIRECT`记得rebuild`fs.img`
- 如果文件系统出了问题也可以考虑rebuild
- `bread()`后记得`brelse()`
- 仅在需要时才分配间接块，可以参考原始实现
- 保证`itrunc`函数能释放一个文件的所有直接/间接块


### debug

"wrong data": 修改block后(即`buf->data`，直接/间接block)记得写回，可以参考原原版用的`log_write`


### result

- 多重索引的实现，不再神秘
	* 通过`buffer cache`层获取磁盘块`bread()`
	* 将磁盘块地址保存为`uint`，且整个块可以看作是连续的，可以使用数组方式访问
- 整体流程abs
	* `buf = bread()`通过缓存层获取磁盘块
	* 根据需要继续遍历磁盘块(`(uint*)buf->data`)来做间接索引

- TODO
	* 为什么一个block的大小是1024byte，而不是4KB呢?


## Symbolic links

添加xv6对符号链接的支持。当打开一个符号链接时，内核会根据地址引用到原文件。实现这个系统调用将会加深对地址查找的理解。

实现`symlink(char *target, char *path)`系统调用，在`path`创建一个引用`target`的符号链接。详情可以查看`man symlink`。

测试：在makefile中添加`symlinktest`然后执行测试


### hints

- 首先创建一个新系统调用号，添加到`user/usys.pl`和`user/user.h`，然后在`kernel/sysfile.c`中实现该系统调用
- 在`kernel/stat.h`中添加**新文件类型**(`T_SYMLINK`)
- 在`kernel/fcntl.h`中添加新标志`O_NOFOLLOW`，这个flag可以用于`open`系统调用，注意不用与其他flag冲突
	* 之后应该能够通过`symlinktest.c`的编译
- 实现`symlink`系统调用创建符号链接。**目标文件不存在系统调用也能执行成功**。你需要在符号链接中保存目标路径。如：inode的数据块。`symlink`系统调用返回0表示成功，-1表示失败
- 修改`open`系统调用来支持符号链接，如果文件不存在，这时才会失败。当有`O_NOFOLLOW`标志时，`open`应该是打开该符号链接文件，而不是链接过去
- 如果别链接的文件也是一个符号链接，需要递归的跟进。如果链接形成了环，需要返回错误代码。可以对递归深度进行限制(e.g. 10)
- 系统系统调用(如link，unlink)则不会根据符号链接，而是操作该符号链接文件本身
- 在这个lab中，你不需要考虑目录的符号链接的情况
	* TODO 可以尝试实现对目录的符号链接


### result

- 对磁盘的操作需要引入事务(`begin_op()`, `end_op()`)，即使简单如xv6都用了事务
	* TODO 深入理解事务
- **什么叫好的抽象**
	* `ip = namei(path)`通过路径获取inode
	* `iread(ip, &data, off, sizeof(data))`，读取inode数据块中的内容
	* `iwrite()`将数据写入inode数据块，与`iread`对应
	* **充分利用连续地址空间来转码**(如将数据块内容转成dirent, syment等)

**hard!!!: 锁策略**

- **`ip = create()`后ip是持有锁的**，记得释放
- 小心锁泄露，如果我这里的实现是`ip = symlookup(ip)`
	* 返回的ip不再是传入的ip，释放锁时需要小心这种情况
- 小心**双重加锁**
	* `readi`要求持有锁，而链接成环又有可能导致双重锁
	* ~~所以我不再使用递归深度判断释放成环，而是判断释放重复持有锁`holdingsleep`。因为我不行违反两段锁协议，从而导致并发bug~~
	* ~~不可以用锁判断，因为有可能是其他程序的正常访问~~
	* 回归用递归深度判断，锁仅在`readi`前后添加
		+ 因为`lock(ip); symlookup(ip); iunlock(ip)`一旦递归起来就无法避免双重加锁
	* 平衡的艺术，想要简单的，不违背两段锁协议的加锁策略有点tricky。**不要尝试在复杂系统中耍小聪明**，但有时候也没办法
- 注意`iunlock != iunlockput`
	* `iunlockput()`会解锁并释放inode，如果不是打开等(后续必用)记得释放，防止inode泄露







