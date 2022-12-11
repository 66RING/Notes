---
title: 使用OCI构建一个容器镜像
author: 66RING
date: 2022-12-09
tags: 
- serverless
- docker
mathjax: true
---

# 使用OCI构建一个容器镜像

> 不同docer启动容器, 了解什么是OCI

https://www.youtube.com/watch?v=hhQ6uc2bp2s


## 什么是OCI

> Container != Docker

- 标准, 类似所有浏览器都使用html/js/css一样, 有了标准后各种容器实现都可以使用根据标注构建的镜像
    * https://github.com/opencontainers
    * runtime-spec
    * image-spec


## image-spec

> https://github.com/opencontainers/image-spec/blob/main/spec.md

- image-layout, 包含以下三个文件/夹内容

解压压缩的文件: `tar xvf -C <output name>`发现就是文件系统, runtime等东西。

那么这么多文件是怎么整合到一个Container当中的? 这就要涉及到UnionFS了


## UnionFS

UnionFS: 让你可以组合多种文件系统, 然后对外暴露统一的一层。可以存在多种实现, 就比如overlay格式(下称overlayfs), 这里就以[overlayfs为例](https://wiki.archlinux.org/title/Overlay_filesystem)。

overlayfs由四部分目录构造: base, diff, merged, workdir。名字可以任取, 后面会通过参数指定

```
$ cd your-overlayfs-name
$ ls
base diff merged workdirr
```

- base目录中的内容就是容器的基础内容, 只读
- diff目录中的内容就是对容器进行操作后变化的内容
- merged就在base + diff结合后对上层暴露的内容
- workdir: The workdir is used to prepare files as they are switched between the layers

因此, 使用overlayfs构建自己的镜像第一步就可以将基础内容拷贝到`base`目录。然后使用mount挂载, 指定参数:

```
$ mount -t overlay -o lowerdir=base1:base2,upperdir=diff,workdir=workdir <anyname-you-want> <merged dir>
$ sudo mount -t overlay -o lowerdir=base:base1,upperdir=diff,workdir=workdir anyname-you-want ./merged
```

- 使用`-t`指定挂载的类型
- 使用`-o`添加文件系统参数
    * `lowerdir`是基础内容, 执行
    * `upperdir`是变动的内容, 可修改
    * `workdir`
    * 最后`<merged dir>`指定合并挂载到哪个文件
    * 然后可以起一个名字`<anyname-you-want>`
- 可以使用`:`组合多个base dir, 如果重复的话右边的base dir会被左边的覆盖, 越左优先级越高

实验`watch -n 3 tree .`, 增删改。当对merged文件夹修改时diff文件夹就会变化, 而base文件夹始终不变。增加文件时diff文件夹中直接新增, 删除时则是在diff文件夹中创建一个同名墓碑文件, 修改文件时则是在diff文件夹中添加一个修改后的文件。

解除挂载后`umount <merged>`, merged文件夹情况, 其他文件夹保留卸载时刻的内容, 下次挂载时就能够恢复

只读和可变分离, 从而确保了文件系统的安全性。


## TODO
