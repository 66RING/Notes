---
title: 为云而生的unikernels
date: 2021-09-18
tags: 
- papers
- os
- cloud
mathjax: true
---

# Preface

记录阅读论文[Unikernel: Library Operating System for the Cloud]，总结、体会

# Abstract

云上时代，很多公司会租用云服务来完成自己的业务。而往往基于多方面的考虑，一台云服务器仅会用于比较单一的功能，如专门为数据库设置的服务器、专门为web设置的web服务器。而云服务供应商在某种意义上仅仅是将系统资源"虚拟化"然后分配给用户使用。

看到云服务这种"单一"功能的属性，unikernel应运而生。不同于传统虚拟机，unikernel为这样云环境做性能、运营、安全、开销等方面的优化。其核心思想就是既然云服务器的使用者仅需要单一的功能，那我们就提供一个功能比较单一的、编译时功能确定的、更轻量更安全的操作系统和配套设施(library)。lib就相当于现在(2021)我们说的"系统生态/软件"，用户可以根据lib随意搭配自己的unikernel。


Unikernels
===========

# 整体架构

# 配置部署


[Unikernel: Library Operating System for the Cloud]: https://dl.acm.org/doi/10.1145/2490301.2451167
