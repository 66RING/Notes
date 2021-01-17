---
title: Docker底层原理
date: 2021-01-17
tags: operate, docker
mathjax: true
---

## Docker底层原理

[Ref](https://docs.docker.com/get-started/overview/#the-underlying-technology)

Docker使用linux内核的一些特性来实现其一些功能


### Namespaces

Docker使用命名空间(Namespaces)来隔离工作区，一个隔离的工作区就称为一个容器。当启动一个容器时docker就会为该容器设置一系列namespaces。

docker使用如下的命名空间:

- The `pid namespace`: 进程隔离
- The `net namespace`: 管理网卡
- The `ipc namespace`: 管理对IPC资源的访问
- The `mnt namespace`: 管理文件系统挂载点
- The `uts namespace`: 隔离内核和版本标识(UTS: Unix Timesharing System).

可以在`/proc/PID/ns`下查看进程的命名空间，观察发现一般情况下宿主机上的进程，进程空间是一样的。而docker启动的进程命名空间与主机隔离。


### Control groups(cgroups)

cgroups用来对一组进程进行资源限制。cgroups让docker可以共享硬件资源，同时也可以做些限制和约束。如你可以限制某个容器的可用内存。

查看系统已有的控制组`ls /sys/fs/cgroup`。使用`cgcreate`创建一个自己的控制组(需要`libcgroup-tools`)，如`cg_test`。设置某个限制`echo 30000 > ./cpu.cfs_quota_us`。然后将进程放入该组`cgclassify -g <cgroup_name> <pid>`


### Union file system

Union file systems, or UnionFS, are file systems that operate by creating layers, making them very lightweight and fast. Docker Engine uses UnionFS to provide the building blocks for containers. Docker Engine can use multiple UnionFS variants, including AUFS, btrfs, vfs, and DeviceMapper.


### Container format

docker将上述namespaces, cgroups, UnionFS组成称为container format。默认的container format是`libcontainer`




