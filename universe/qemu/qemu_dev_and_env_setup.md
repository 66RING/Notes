---
title: QEMU虚拟机开发与环境配置
date: 2021-07-29
tags: 
- qemu
mathjax: true
---

# QEMU虚拟机开发与环境配置

QEMU虚拟机开发与环境配置大体分为以下三个步骤：

- qemu虚拟机开发
- 准备虚拟机运行所需环境
- qemu运行参数配置


## 虚拟机开发

虚拟机开发可以从最简单的图灵机模型开始，创建内存、创建CPU，再在此基础上添加需要的设备。需要注意的是图灵机模型不包括屏幕回显功能（毕竟有输出才方便debug嘛），所以含有CPU，内存，serial设备的机器才是人类能够交互的**最小虚拟机模型**。

之后就能根据内核启动的输出信息在虚拟机中补充相应的设备来不完虚拟机了。具体详情以及踩坑可以参考[这篇文章](https://github.com/66RING/Notes/blob/master/universe/qemu/powerpc_sim.md)。如注册CPU reset事件、加载内核、配置设备树等。


## 准备虚拟机所需环境

准备虚拟机所需环境包括：

- 虚拟机
- 适当的linux镜像
- 根文件系统


### 创建适当的linux镜像

为何说要创建适当的镜像？因为linux内核支持很多设备，为了不让内核过于庞大linux将各种功能模块化，用户可以根据需要选择是否编译到内核中。如linux对ext文件系统、jffs2文件系统、flash驱动、ramfs等的支持都是可选的。用于可以用`make menuconfig`进入配置界面进行选择。

总结一下就是：根据具体的硬件，勾选相应的linux内核选项，使linux提供软件支持。具体硬件与linux内核选项的对应关系通过查阅官方文档获悉。如硬件使用powerpc架构e500核心的系列cpu，则勾选linux内核选择中`Processor support --> Processor Type --> Freescale 85xx`。

linux预置了多种平台的支持Platform support，这些平台即包括硬件的Freescale系列开发板`Platform support --> Freescale Book-E Machine Type --> Freescale xxxx`，也包括软件的qemu虚拟机开发板`Platform support --> Freescale Book-E Machine Type --> qemu generic e500 platform`。所以使用Freescale 85xx选项让linux提供了基本的支持后，可以进一步选择linux预置的平台支持，或仅勾选需要的组件，甚至在linux源码中添加自定义内容。

这里提供一种配置思路：

- 1. `make defconfig`加载默认配置
	* linux提供了多种默认配置，可以在`arch/TARGET/configs`目录中查看
		+ TARGET指代目标指令集架构
- 2. 使用`make menuconfig`在默认配置的基础上根据需求进一步修改

如我要创建一个简单的PowerPC e500的虚拟机，查看文档发现linux为e500提供的默认配置文件叫`corenet`，在`linux/arch/powerpc/configs/`目录下果然有个相关的配置`corenet_base.config`。所以通过`make`加载配置：

```
make corenet_base.config
```

之后进行进一步配置，查阅文档发现，要制作qemu能够运行的e500镜像，需要打开e500支持的内核选项：

使用`make menuconfig`进入配置界面

```
make menuconfig
```

勾选`QEMU generic e500 platform`选项。

```
Platform support -> 
	Freescale Book-E Machine -> 
		QEMU generic e500 platform
```

我打算使用initrd简单的启动镜像，所有还有开启initrd的支持，勾选：

```
General setup ->
	Initial RAM filesystem and RAM disk (initramfs/initrd) support
```

保存退出编译镜像

```
make
```

编译生成的镜像文件一般生成在linux源码根目录和`linux/arch/TARGET/boot`目录。常见镜像的有：

- vmlinux，未压缩的内核ELF文件
- vmlinuz，vmlinux的压缩文件
- zImage，vmlinuz经过gzip压缩后的文件


#### 注意事项

Linux提供了e500mc核心的支持，在`processor support -> e500mc support`处，开启后只能运行使用了e500mc核心的虚拟机，e500不行。


### 创建根文件系统

这里使用busybox创建根文件系统。busybox提供了一些基本的命令行工具，如mount、sh等。利用busybox提供的工具可以完成linux文件系统的挂载。

下载buxybox源码：

```sh
wget https://busybox.net/downloads/busybox-1.33.1.tar.bz2
```

解压并设置

```sh
cd busybox
make menuconfig
```

勾选：

```
Setting ->
	Build static binary (no shared libs)
```

编译安装

```
make ARCH=powerpc CROSS_COMPILE=powerpc-linux-gnu-

make install
```

由于我要为powerpc架构的linux准备根文件系统，所有架构指定为powerpc，即`ARCH=powerpc`，工具链使用事先准备好的交叉编译工具`CROSS_COMPILE=powerpc-linux-gnu-`，交叉工具安装配置见第二章。在`make install`默认将间编译生成的文件放置在`_install`文件夹。

```sh
$ ls ./_install
bin  linuxrc  sbin  usr
```

通过initrd启动内核的方式还需要一个init程序，init程序负责完整文件系统的挂载等的工作，但这里是最小化实现只做sh的打开。init程序如下：

```sh
#!/bin/sh

echo "###############################"
echo "######### Hello World #########"
echo "###############################"

exec /bin/sh
```

init程序只打开了shell，如果要挂载完整linux文件系统，可以在进入shell后手动完成，也可以写在init脚本中进行自动化。

最后，使用initrd启动需要将根文件系统通过cpio打包，进入`_install`文件夹：

```sh
$ cd _install
$ ls
bin  linuxrc  sbin  usr  init
```

使用如下命令进行打包：

```sh
find . -print0 | cpio --null -ov --format=newc | gzip -9 > initrd.gz
```

至此根文件系统准备完成。


## qemu运行参数配置

假设通过第一章我们创建了一个名叫p2020的虚拟机，编译安装qemu后得到`qemu-system-ppc`二进制文件。通过第二章，我们又获得了linux镜像`vmlinux`和根文件系统的initrd文件`initrd.gz`。于是我们可以通过如下命令启动虚拟机：

```c
qemu-system-ppc  \
  -M p2020 \
  -m 512M \
  -initrd ./initrd.gz \
  -kernel ./vmlinux \
  -append "rdinit=/init"
```

- `-M p2020`指定我们运行的虚拟机为p2020
- `-m 512M`为虚拟机提供512M内存
- `-initrd ./initrd.gz`指定使用的initrd文件
- `-kernel ./vmlinux`指定使用的镜像文件
- `-append`添加内核启动参数，这里`rdinit=/init`告诉内核使用的init程序的位置。这步可以省略，因为内核默认会搜索根目录、`sbin`、`bin`目录下名为`init`的文件作为init程序


# 交叉工具的安装与配置

编译交叉工具是个费时费力的过程，好在很多厂商为我们提供了编译好的交叉工具我们可以直接使用。常见的交叉工具"供应商"有：

- [DENX ELDK](http://www.denx.de/wiki/DULG/ELDK)
- [Bootlin](http://toolchains.bootlin.com/)
- [musl.cc](https://musl.cc/#binaries)
- [Linaro (ARM)](https://wiki.linaro.org/WorkingGroups/ToolChain)


## DENX提供的工具链安装

这里我们首选DENX ELDK家的交叉工具进行安装，原因是：

- 1. DENX提供的工具链版本较低，如gcc是4.3版本的。而我们要编译的是内核版本较低的linux，在使用高版本工具链时编译报错
- 2. DENX是一家专注嵌入式实时操作系统的大公司，著名的uboot就是他们开发的

首先去DENX官网下载对应工具：[The Embedded Linux Development Kit (ELDK)](https://ftp.denx.de/pub/eldk/)。这里eldk使用的版本是5.6，需要powerpc架构的工具链，所以下载[eldk-5.6-powerpc.iso](https://ftp.denx.de/pub/eldk/5.6/iso/eldk-5.6-powerpc.iso)的镜像。

以下是工具链安装过程：

挂载`eldk-5.6-powerpc.iso`，这里挂载到`/mnt`目录

```sh
$ mount /path/to/eldk-5.6-powerpc.iso /mnt
$ cd /mnt
```

执行安装

```sh
./install.sh -d /path/to/install_dir -s toolchain-xenomai-qte -r qte-xenomai-sdk powerpc
```

- `-d`指定工具链的安装目录。如果不指定，默认安装到`/opt/eldk`
- `-s`表示SDK使用的镜像源
- `-r`表示目标使用的RFS镜像源
- 最后一个参数表示安装的目标指令集架构，这里是`powerpc`

更多参数详情查看`./install -h`

工具链就安装到了`/path/to/install_dir`目录了，使用时将需要的二进制文件添加到环境变量即可，如：

```
export PATH=$PATH:/path/to/install_dir/powerpc/sysroots/i686-eldk-linux/usr/bin:/path/to/install_dir/powerpc/sysroots/i686-eldk-linux/usr/bin/powerpc-linux:/path/to/install_dir/powerpc/sysroots/powerpc-linux/usr/bin/powerpc-linux
```

正确安装工具链并配置好环境变量后可以打印工具来版本信息：

```
powerpc-linux-gcc -v
```


## musl.cc提供的工具链安装

- 下载需要的[工具链](https://musl.cc/#binaries)
- 解压缩
- 添加环境变量


## Bootlin提供的工具链安装

- 下载需要的[工具链](https://toolchains.bootlin.com/)
- 解压缩
- 添加环境变量











