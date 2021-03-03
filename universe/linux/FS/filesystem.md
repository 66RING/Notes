---
title: linux文件系统
date: 2020-12-05
tags: 
- linux
- filesystem
mathjax: true
---

# 文件系统原理

为了高效组织、管理和使用磁盘上的数据，需要独立的文件系统。


## 文件系统的组成

通常磁盘上的数据是要与内存交互的(内存映射)，而内存一个页的大小是4KB，所以磁盘一般按照4KB进行划分。一个划分就称为一个block。

一个文件本质上是一些字节的集合，这些字节就是文件的 **user data**，同时引出以下概念

- **meta data**，文件本身数据外的控制信息，如访问权限、大小、创建时间等
    * 可以理解成关于user data的data
    * meta data存储的数据结构就是inode

为了追踪inodes和data blocks的分配和释放情况，最简单的办法就是使用bitmap，分别维护记录inodes的bitmap和记录data block的bitmap。如果每个block为4kb，则bits数量为4096x8。

**superblock**，包含一个文件系统所有的控制信息，如文件系统有多少个inodes和data block，inode的信息从第几个block开始等，可能还有一个区别不同文件系统类型的magic number。。因此，superblock可理解为是文件系统的meta data。


## 文件寻址

用于存储inode的block组成一个 **inode table** ，

一个文件需要占据若干个blocks，这些blocks可能是磁盘上连续分布的，也可能不是。所以，比较好的办法是使用指针，指针存储在inode中，一个指针(这里的指针是一个block编号)指向一个block。假设一个inode最多包含12个指针，那么不大于48kb的文件一个inode就能搞定。

如果文件大小超过了一个inode能包含的文件大小，可以由inode先指向一个block，这个block再指向分散的data block。如果一个指针占4字节，则一个中间block能存储1024个指针，以此类推二级三级。

对于只使用block指针的方式存在一个问题，就是需要较多的meta data。而实际大多数文件的体积都很小，meta data相对user data的占比就较大。

另一种实现是使用一个block指针加上一个length来表示一组物理上连续的blocks(称为一个 **extent** )


## 目录和路径

目录可以看成一种特殊的文件，它的user data存储的是普通文件的inode

要查找一个文件就可以总根目录“/”开始，根目录的inode是事先知道的，根据其user data找到对应的文件或目录。以此类推，直到找到对应文件。


# 零碎

## inode

inode不存储数据内容，存储元数据信息，如类型、权限、拥有者、时间、链接数、文件内容位置等

当文件系统(以下简称fs)格式化好后，inode会以数组的方式存储，每个元素是一个inode，inode-index就是对应的下标。fs除了生成inode数组还有生成一个map映射文件名和inode-index，使用文件名去map找对应的idx，就可以在inode数组里找到对应的文件元信息。

inode中文件内容位置记录的是块的下标

由于inode数组是有初始化大小的，所有存在inode用完的情况，即在有很多零碎文件的情况下。

### 查看inode情况

- `df -i`可以查看inode数组，以及数组元素使用/剩余情况
- `ls -i`可以查看文件夹文件对应的inode-index
    * 文件访问的过程中，先找到文件的idx，然后去对应的inode数组中找元数据



