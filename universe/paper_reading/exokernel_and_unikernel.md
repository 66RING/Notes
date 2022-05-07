---
title: 外核架构Exokernel和Unikernel 
author: 66RING
date: 2021-09-18
tags: 
- papers
- cloud
- os
- system
mathjax: true
---

# Abstract

看到云服务这种"单一"功能的属性，unikernel应运而生。不同于传统虚拟机，unikernel为这样云环境做性能、运营、安全、开销等方面的优化。其核心思想就是既然云服务器的使用者仅需要单一的功能，那我们就提供一个功能比较单一的、编译时功能确定的、更轻量更安全的操作系统和配套设施(library)。lib就相当于现在(2021)我们说的"系统生态/软件"，用户可以根据lib随意搭配自己的unikernel。


# Preface

内核对硬件资源进行抽象，抽象必然代理损失。因为**只有应用才知道自己需要什么**，没有一种抽象是全能的。


# Overview

- Exokernel将管理和保护分离，**内核只复制资源的保护，不负责资源的管理**
- Exokernel要实现的保护功能：
	* 追踪资源所有权
	* 保证资源的保护(拥有权)
	* 回收资源的访问权
- 资源与应用绑定，内核需要回收资源的能力防止问题应用
	* 可用性: 
	* 隔离性: 防止相互


# Exokernel + LibOS

exokernel不再提供抽象，难道应用要真正从0开发？答案是**exokernel将硬件抽象以库的方式提供给用户**, 即**LibOS**。用户可以根据需要定制地选择所需的库或者自己实现。

即：高度定制与通用LibOS的tradeoff


# Unikernel架构

思路: 一个应用单独运行一个虚拟机，则应用直接运行在虚拟化提供的内核态(保证了隔离)，而原本的操作系统只剩下服务该应用的功能，成为Unikernel(每个应用不同)

- 虚拟机 = 应用
- 虚拟机底层基于exokernel
- LibOS = Unikernel
- 虚拟机保证隔离性
	* 容器
	* 每个容器运行定制的Unikernel提高性能


# 亮点与不足

- 亮点
	* 极致微内核
	* 无抽象
	* need no full feature and portable
- 缺点
	* 应用决定，开发难度大
	* 定制多，维护难，生态难统一


# 收获

- 只有用户才知道自己需要什么，比如DBMS中很多bypass。
- 抽象的代价
- 虚拟化提供退unikernel的隔离
- 以前有，为何不做?不火?better solution?


# References

- [Unikernel: Library Operating System for the Cloud](https://mort.io/publications/pdf/asplos13-unikernels.pdf)
- [Exokernel: An Operating System Architecture for Application-Level Resource Management](https://pdos.csail.mit.edu/6.828/2008/readings/engler95exokernel.pdf)


