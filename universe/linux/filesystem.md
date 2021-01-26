---
title: linux文件系统
date: 2020-12-05
tags: 
- linux
- filesystem
mathjax: true
---

## filesystem

### inode

inode不存储数据内容，存储元数据信息，如类型、权限、拥有者、时间、链接数、文件内容位置等

当文件系统(以下简称fs)格式化好后，inode会以数组的方式存储，每个元素是一个inode，inode-index就是对应的下标。fs除了生成inode数组还有生成一个map映射文件名和inode-index，使用文件名去map找对应的idx，就可以在inode数组里找到对应的文件元信息。

inode中文件内容位置记录的是块的下标

由于inode数组是有初始化大小的，所有存在inode用完的情况，即在有很多零碎文件的情况下。

#### 查看inode情况

- `df -i`可以查看inode数组，以及数组元素使用/剩余情况
- `ls -i`可以查看文件夹文件对应的inode-index
    * 文件访问的过程中，先找到文件的idx，然后去对应的inode数组中找元数据



