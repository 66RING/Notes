---
title: GiantVM论文笔记
date: 2021-10-23
tags: 
- papers
- distributed
mathjax: true
---

# Abstract

传统的：租用多台云主机，每台云主机安装OS，在使用一些分布式框架实现在这些OS的相互配合。这么做存在诸多问题，一方面分布式框架的使用和设计会增加开发者的门槛，另一方面，在超大规模的分布式环境下分布式的复杂性、操作难度都会进一步增加。

于是为了克服分布式系统带来的问题，隐藏分布式系统的复杂性，人们首先提出了SSI(Single System Image)，意图实现操作多个分布式节点像操作单一节点一样简单。但这个方案同样存在一些问题需要克服：传统的SSI方案需要修改操作系统，修改操作系统就要涉及到OS生态的移植问题(如驱动程序根据POSIX标准的重新设计等)。一方面一种操作系统就带来如此大的工作量，另一方面有一些驱动的闭源的。

因此这篇论文提出在hypervisor层支持SSI的方案，这样设计无需修改guest OS就能实现SSI。

> 计算机中的一切问题都可以通过加一层抽象解决


# Preface

当今的虚拟化方案有两种类型: type-I, type-II。type-I是直接运行在裸机上的hypervisor(如xen)，而type-II是运行在host OS上的。type-I虽然能获得更好的性能，但是无法利用host OS提供的抽象。GiantVM(QEMU)使用的是type-II的方案。

对于虚拟化可以做出如下抽象：

1. CPU虚拟化
2. 内存虚拟化
3. I/O虚拟化

GiantVM就需要对上述内容进行一些分布式的实现，这里解释用到的一些技术：

1. CPU虚拟化
	a. 以来硬件平台的支持如Intel的VMX和AMD的SVM
2. 内存虚拟化
	a. 内存虚拟化一般通过shadow page table实现，shadow page table就是用来做虚拟机物理地址GPA到宿主机虚拟地址GVA映射，为虚拟机提供内存虚拟化环境的
	b. 如Intel中的实现就是EPT(Extended Page Table)
3. I/O虚拟化
	a. Intel中引入IOMMU技术，让虚拟机能够直接利用宿主机的硬件资源
	b. 如虚拟机直接使用GPA发起DMA，而IOMMU的作用就是自动地将GPA翻译成HPA
4. KVM
	a. linux中为硬件加速虚拟化资源的抽象，可以客户端通过KVM来来使用CPU、内存、IO等虚拟化资源
5. DSM(Distributed Shared Memory)
	a. 分布式hyperviosr的核心技术，DSM可以将多个节点上的物理机的内存整合成一段连续的虚拟内存
	b. 这段虚拟内存如果供hypervisor使用，那么就实现了分布式hypervisor中资源共享的抽象
6. RDMA(Remote DMA)
	a. 更低延迟、更高吞吐量的网络技术，分为one-side和two-side两种类型
	b. one-side可以让本地cpu直接发起远程的DMA，而不需要经过远程CPU
	c. two-side就比较类似传统的socket等


# 正文内容

## 设计

### 分布式CPU

通过实现处理器间中断(Inter Processor Interrupt)实现cpu间通信。通过修改APIC是模拟完成。


### 分布式内存

使用的DSM基于Ivy协议(protocol of Ivy)。有点类似简化版的MESI协议，仅有MSI三种状态：

- Modified: page是当前节点独占的，任意读写
- Shared: page是节点间共享的，**读保护**
- Invalid: 失效的page，需要重新从最新的节点拷贝

读写Invalid或写Shared会触发page fault: 读的化要在hypervisor层找到owner再读，写的话找到owner写，然后让其他老的节点(Shared)失效。

### 分布式IO

在hypervisor层模拟设备发起的中断和cpu发起的中断，然后再利用host端的通信实现。


## 具体实现(Overview)

### 分布式CPU

- dummy APIC
	* 在本地初始化vCPU和他们的PIC(dummy PIC)，以便能感知到
	* point -> point: 广播

### 分布式内存

TODO: 重看

- DSM
	* 映射管理
	* EPT使用

### 分布式IO

- 主要实现：int from device和int access to device
	* 兼容legacy和modern(PIO, MMIO, MSI)的方式
		1. KVM退出点判断
		2. MSI: 广播

TODO: 没有实现，比较有难度


### 时间同步

TODO: 重看

可以利用硬件特性(Time Stamp Counter (TSC))和KVM

# 实验设计

# 亮点与不足

## 亮点

可以说是各种技术的一种基本应用吧，毕竟是原型机。虽然存在类似的设计ScaleMP

## 不足

# What's more

1. 内存一致性问题
2. Fault Tolerance问题

## 内存一致性模型

TODO: 重看

考虑sequential consistency模型和TSO模型

## FT

GiantVM没有实现，但是分布式系统中很重要的一种技术，难点在于GiantVM在hypervisor层，而需要FT往往的通过冗余节点备份的形式。所以需要进一步抽象SSI和分布式hypervisor该如何设计。


# 思考

1. 一致性和FT始终是分布式需要考虑的内容(当然还有吞吐量、bottleneck等诸多问题需要考虑)
2. 从最小化开始
3. 没准可以想GFS一样，通过观察实际业务需要后做进一步的定制优化设计


