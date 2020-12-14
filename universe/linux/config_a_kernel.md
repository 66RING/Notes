---
title: 如何配置linux内核
date: 2020-11-29
tags: linux, kernel
---

## 如何配置linux内核

linux内核提供许多自定义的选项

因此你可以根据自己喜好定制的编译linux内核。这里以在centos8中编译centos8的内核为例。


### 准备内核源码

> [官方文档](https://wiki.centos.org/zh/HowTos/I_need_the_Kernel_Source)

centos的内核使用rpm管理，所以在真正获得内核前你还需要使用rpm把内核安装到机器上。可以在`https://vault.centos.org/`下载对应的内核rpm包(**内核源码版本一定要和系统内核版本一致**)。如

```sh
wget https://vault.centos.org/8.3.2011/BaseOS/Source/SPackages/kernel-4.18.0-240.1.1.el8_3.src.rpm
```

然后使用rpm安装内核源码

```sh
rpm -i kernel-4.18.0-240.1.1.el8_3.src.rpm
```

这里可能会提示

```
Warning: user mockbuild does not exist. using root
```

那就创建一个mockbuild用户

```sh
useradd mockbuild
```

然后继续安装

```
rpm -i kernel-4.18.0-240.1.1.el8_3.src.rpm
```

这时会在根目录会得到一个rpmbuild目录。在`~/rpmbuild/SPECS`中进行`rpmbuild`

```sh
cd ~/rpmbuild/SPECS
rpmbuild -bp --target=$(uname -m) kernel.spec
```

这时可能会碰到依赖问题，需要安装很多依赖包，安装依赖后再次尝试`rpmbuild`

```sh
yum install audit-libs-devel binutils-devel elfutils-devel java-devel kabi-dw libcap-devel libcap-ng-devel llvm-toolset ncurses-devel newt-devel numactl-devel openssl-devel pciutils-devel python3-devel python3-docutils  xz-devel zlib-devel
```

这样一来就得到了内核源码了，在`~/rpmbuild/BUILD`目录下


### 自定义内核

在源码中使用`make menuconfig`可以进行内核选项的设置，然后配置结果会保存在`.config`文件中，make时会根据`.config`的结果进行编译。当然也可以载入现成的配置文件。如本机(centos8)的内核配置文件就在`/boot/config-$(uname -r)`里。拷贝到源码所在目的`.config`文件，然后选择可以加载配置或者`make menuconfig`是选择LOAD加载指定文件

```sh
cp /boot/config-$(uname -r) ./.config
make menuconfig
```

`menuconfig`里面可以自己选择也可以使用默认或加载配置。做好你的选择后就可以开始编译了

```sh
make -j 8
# -j [jobs]
# 指定同时能够运行的jobs(命令)的数量，如果-j选项没给出参数则不限制jobs数量
```

安装模块，linux的模块是动态加载的，故模块安装和内核安装相互独立

```sh
make modules_install
```

安装内核，重启后就是新的内核了

```
make install
```

