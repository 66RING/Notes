---
title: 虚拟化综述
author: 66RING
date: 2022-05-08
tags: 
- virtualization
mathjax: true
---

# summary

- "总是从简单的分层抽象开始，逐渐打破分层，然后融合"
- 虚拟化 = fake interface, thanks to layered
- 加速法
	* 硬件打配和
	* bypass VMM
	* 考虑拷贝开销：共享内存，减少拷贝
- 要隔离那再建"转换表"


# Preface

黑客能如何破坏你的电脑：`fork boom`就让你的进程永远无法及时调度。

就算虚拟机内有1亿个进程，也不会把host os的队列撑爆。因为对于host虚拟机仅对应vCPU个线程，不会把调度队列撑爆。


# 理论

- 本质是提供不同层次的接口
	* ISA
	* ABI
		+ plus OS system call
	* API
	
VMM/Hypervisor = 向上提供不同的多个ISA

- type 1
	* VMM直接运行在裸机，此VMM可以管理多个VM
	* 定制，挑硬件
	* 性能比较好
- type 2
	* OS上运行VMM
	* 有OS用，用户友好
	* **市场占有率**
	* 利用host的大多数功能
		+ 调度交给host OS(暴露出vCPU thread)
		+ 物理内存管理
		+ 驱动


## 几种方式

- 解释执行
	* 指令调相应函数模拟
	* 敏感指令不下陷
	* 支持多ISA
	* 照着硬件文档，模拟就行(nju pa)
- 二进制翻译
	* 批量翻译，**缓存**取已翻译的指令
	* 读真寄存器 -变为-> 读虚拟机的虚拟出来的寄存器
	* 不能处理Self-modifying Code
		+ JIT, JVM
			+ e.g: jvm将二进制翻译成native code，jvm然后运行的时候会修改自己生成的代码。那么该修改自己的代码还是缓冲区里的代码(代码补丁)呢？
	* 中断粒度大，basic block 为单位
- 半虚拟化
	* **修改 GuestOS(驱动) Hypercall 替换 敏感指令**
	* 虚拟机中知道它会执行性敏感指令，那就修改guest os(驱动)，让使用敏感指令的地方调用VMM提供的接口hypercall
	* 需要修改OS，闭源系统难应用
- 硬件虚拟化
	* intel VT-x
		+ 专门为虚拟机创建一个non-root mode"特权级"
		+ ring 0, ring 3, OS运行在ring 3，在ring 3中又嵌套(ring0, ring3)，专门给VMM，嵌套的ring 0, ring3就叫non-root mode
		+ VMCS结构用于维护"上下文"
		+ ring0 -> ring3 特权级级高 -> 低
	* ARM VHE


## 基本流程

1. 捕获所有系统ISA trap
2. 根据具体指令实现相应虚拟化
3. 回到虚拟机


## QEMU/KVM架构

- QEMU提供很多虚拟设备的支持，比较成熟
- 内核模块，CPU驱动(提供服务给用户态)
- QEMU/KVM打配合
	* KVM处理不了vmexit，进QEMU设备模拟

使用`/dev/kvm`文件接口与KVM通信，`ioctl()`传递命令


```
QEMU
| ioctl()
v
/dev/kvm 	  (VM entry, 进内核..., vm exit, case exit reason)
| 				 ^
v				 |
KVM		->    Hardware
```


### CPU虚拟化

`ioctl(KVM_RUN)`: "保存VM现场"，"加载"

- ARM
	* eret进入EL2，道理类似内核态切用户态
- x84
	* 使用VMCS结构保存，硬件自动同步状态

KVM不单单可以处理CPU相关还可以处理其他，如内存等。所以VM exit后先看看KVM内不能处理，不能处理再让QEMU处理。


### 内存虚拟化

> 3种地址，2种页表

虚拟机希望0 base的内存，我们不可能满满足每个虚拟机真正的物理内存，而且直接物理内存要考虑到安全性问题。

面临的挑战：GVA -> GPA(HVA) -> HPA太多转换了，开销较大

- 影子页表Shadow Page Table
	* 就是说GVA(一个页表嘛)直接映射到HPA，即直通虚拟机内的页表
	* GVA -> (OS page) -> GPA -> HPA直接变成: GVA -> HPA
	* 需要为每个虚拟机的每个进程维护(通过软件方式)一个影子页表，维护的页表就会很多
	* "虚拟机内进程的页表直接作为host的进程页表"
- 直接页表Direct Page Table
	* "虚拟机直接操作物理地址", "告诉虚拟机不连续的物理地址，自己维护去"
	* 但是客户的虚拟机知道自己的物理地址，那也能知道别人的物理地址，欠安全
- **硬件虚拟化**
	* 本质就是**再加一层翻译** ，维护GVA到HPA的映射
		+ Intel Extended Page Table
		+ ARM 第二阶段页表
	* 显然缺点是内存访问次数增加，不过对于开发来说不用维护影子页表了
		+ 24次访存：GVA有4级页表，每级拿到的GPA还要经过EPT(或这说第二阶段页表)翻译
	* 太慢了**加TLB**
		+ 保存GVA -> GPA和GPA -> HPA的结果
- 处理缺页异常
	* guest中的和VMM的


### IO虚拟化

- 所有vm直接访问的问题
	* e.g. 网卡，相同的mac地址，那应该发送给哪个VM呢？
	* 安全性问题, DMA后又设计内存安全

- 目标
	* 功能正确性
	* 隔离性
	* 物理设备资源利用率


#### 设备模拟

* 根据规范把接口都实现了
* qemu在中间做拦截
* 发送情况
	+ PIO情况可以通过捕获中断实现
	+ MMIO情况我们可以设置invalid，然后再handlepagefault时处理
* 接收情况
	+ qemu注入虚拟中断

```
          ┌────────────────┐        ┌──────────────────────────┐
          │     VMM        │        │           VM             │
          │ QEMU   ◄───────┼────────┼──┐                       │
          │           4    │        │  │                       │
          │  │             │        │  │                       │
          │  │             │        │  │                       │
          │  │             │        │  │                       │
          │  │             │        │  │        Network Stack  │
          └──┼─────────────┘        │ ┌┴──────┐       │        │
             │         ▲            │ │buffer │       ▼        │
    5 syscall│         │            │ └───────┘ Network Driver │
             │         │            │                 │        │ 1
  ┌──────────▼─────────┴───┐        └─────────────────┼────────┘
  │Linux   Driver     KVM  │ 3                        │
  └──────────┬─────────▲───┘                          │VM exit   2 trap
             │         │                              │
             ▼         │                              ▼
                     ┌─┴────────────────────────────────────────┐
            HW       │  Hardware virtualization                 │
                     └──────────────────────────────────────────┘

```


1. VM以为自己是真机，尝试访问硬件
2. 被硬件虚拟化截获到VM的方法
3. KVM发现它无法解决，交由QEMU模拟
4. 因为VM的内存是QEMU申请的，QEMU VM的所有内存，那么QEMU就能截获数据然后"代发"
5. QEMU用户态"代发"使用syscall与硬件交互


#### 半虚拟化

1. **绕过设备模拟**(上图中的Network Driver)
	- 因为经过网络协议栈后其实已经可以发送了, 后面又得过host机的驱动
2. **共享内存减少拷贝**
	- virtqueue

- 虚拟机内运行前端驱动(front-end)
- VMM内运行后端驱动(back-end)
- VM内经过网络协议栈后就可以直接发送了，不过这时不是经过qemu模拟出来的VM内的driver，而是交给front-end，然后front-end以通过**共享内存**与back-end交互，back-end直接扔给host的driver发送
- front-end -> (bypass qemu) -> back-end

```
         
         ┌────────────────┐               ┌───────────────────┐
         │      VMM       │               │        VM         │
         │ QEMU           │               │                   │
         │       back-end ◄──Virtqueue    │                   │
         │        │       │               │                   │
         │  ┌─────┘       │          ▲    │                   │
         │  │             │          │    │                   │
         │  │             │          │    │    Network Stack  │
         └──┼─────────────┘          │    │          │        │
            │         ▲             2│    │          ▼        │
   5 syscall│         │              └────┼─ front-end Driver │
            │         │                   │          │        │ 1
 ┌──────────▼─────────┴───┐               └──────────┼────────┘
 │Linux   Driver     KVM  │ 4                        │
 └──────────┬─────────▲───┘                          │VM exit   3 Hypercall
            │         │                              │
            ▼         │                              ▼
                    ┌─┴────────────────────────────────────────┐
           HW       │  Hardware virtualization                 │
                    └──────────────────────────────────────────┘

```

- **virtio**: 标准化半虚拟化IO框架
	* 通用前端抽象
	* 标准化接口
	* 代码的跨平台重用
	* **virtqueue**: VM和VMM之间传递IO请求的队列
		+ descriptor table
			+ 描述前后端共享的内存
			+ 链表组织
		+ available ring
			+ ring entry指向一个descriptor链表
		+ used ring
			+ 已用的descriptor的索引
	* 好处：
		+ 跳过模拟
		+ 针对性做batch
	* 缺点：改内核


#### 设备直通

- 问题1 ：设备间的隔离性。根源：外设直接看到HPA。
	* 对于内存，我们有EPT，EL2等机制让外设(e.g. DMA)直接看到GPA，从而隔离
	* 对于网卡，我们有IOMMU，IOMMU要做映射，那么这个映射用的table就可以用stage 2的table。从而网卡看到GPA

- 问题2: 设备独占问题
	* Single Root Io Virtualization(SRIOV)
		+ 在设备层实现虚拟化，如一个网卡有自带多个虚拟网卡


#### 中断虚拟化

本质就是VMM知道VM的完整的状态机，通过修改寄存器(具体见手册)可以注入中断

- 触发vm exit的中断虚拟化
- 不触发vm exit的中断虚拟化
	* 硬件打配合


