---
title: Persistent Memory软硬件结合方向的调研
author: 66RING
date: 2022-06-26
tags: 
- paper
mathjax: true
---

# abstraction

基于新硬件的存储系统性能优化


# 已经进行了哪些研究

> 有哪些创新性方法
> 取得了哪些结果
> 为什么对这个方向感兴趣


通过这段时间对实验室工作的调研和论文阅读我大概知道了实验室的战略方向和该方向的一些前沿研究。

首先我感受到研究的基本思路是: 非易失型存储设备PM(Persistent memory)的出现模糊了当前系统架构"极端存储"和"极端内存"的界限, 需要对这种新硬件提供支持跟进的同时还要针对性地设计新抽象和算法。

我看到实验室在PM(Persistent memory)生态建设方面完成了一系列布局，从基础设施的搭建到算法/架构的针对性优化，再到应用实践的。我选出了如下我觉得比较有代表性的创新成果

- 基础设施
	* VMFS: VMFS:一种持久性内存统一管理系统[^1]
		+ 提供统一管理内存和存储, 利用PM硬件特性, 优化了(memory)中间层的引入带来的性能损失
	* PCOW: Towards Virtual Machine Image Management for Persistent Memory[^2]
		+ 完善云平台对PM的支持, 减少IO操作导致的VM Exit从而提升性能
- 算法/框架的优化
	* SLO-Aware IO框架: A Thread Level SLO-Aware I/O Framework for Embedded Virtualization[^3]
		+ 提出一种半虚拟化框架，绕过操作操作系统让底层设备感知
	* SwapKV: A Hotness Aware In-memory Key-Value Store for Hybrid Memory Systems[^4]
		+ 利用PM设备的高速低价的特性, 提出一种内存存储统一管理系统
- 应用实践: 主要利用PM随机寻址和大容量的特性
	* An NVM SSD-based High Performance Query Processing Framework for Search Engines. IEEE Transactions on Knowledge and Data Engineering [^5]
	* 新型存储设备上重复数据删除指纹查找优化[^6]

基础设施方面我想从PCOW镜像格式, VMFS文件系统和SLO-Aware IO框架进行说明表达我的感受。定制化和混合是当前的主要趋势，PM的出现就模糊了memory和storage的界限，而当前系统架构中的设计多是针对"极端的存储"和"极端的内存"进行的，所以需要研究对"混合模式"的抽象。以VMFS为例, 考虑到PM的存储特性和内存特性

VMFS和SwapKV提供了一种新的抽象思路并给出了一些问题的解决方案: 利用PM的存储特性和内存特性, 如果PM即是存储又是内存那么就能够减少分层带来的数据拷贝开销，实现外存和内存之间的零拷贝。而且在SwapKV存储系统中, 我们实验室也考虑到了实际发展问题，通过操作映射的方式让DRAM操作与PM操作兼容，扩大市场。利用文件块的异步操作来解决PM设备速度略差于DRAM的问题。我们将分层打破或者绕过, 会不会有潜在的安全问题和可扩展性问题呢? 拿数据库为什么不简单使用mmap做缓存管理的原因作为例子，mmap会自动写回可能会破坏事务的安全性, 解决办法是使用影子页。同样的这里我们的PM即做存储又做内存, 就相当于直接写到存储, 那么为了事务的安全性开发人员就要重新适应新设备的模式。而且在可扩展性问题方面，我们的操作系统应该抽象PM设备, 如果存储当内存访问，那么会不会导致大量的CPU cache的污染然后又因为CPU要保证缓存一致性，导致CPU间频繁通信导致可扩展性问题。上面提到的问题就是说, 细节问题也许可以解决，但是引入了额外复杂度, PM设备能不能做好抽象是关键。

PCOW镜像管理系统和SLO-Aware IO框架则从PM生态建设方面入手的文章，让云平台也支持上PM设备并利用PM设备特性。我想说说我的体会，在数据库系统中，我们常常会绕过操作系统提供的"通用支持"，如mmap等。不过在云计算时代，我们的服务往往是建立在云服务提供商的云平台上的，那么我们为了性能和安全绕过了操作系统是不是也应该绕过云平台? 或者云服务提供商应该针对性地提供支持? 根据我的理解和本次调研，我感受到解决方案是: 抽象和半虚拟化。我们将云计算服务抽象成IaaS、PaaS和SaaS等类型以满足不同用户的不同需求，那么考虑到PM设备的特性, 我们是否可以提出新的抽象呢? PCOW镜像管理系统论文给了我启发，"Container as a Service"的优势是轻量和启动快, 那么PM设备利用其本身的硬件属性就能解决启动速度问题, PCOW系统由从镜像管理方面解决了轻量化的问题，所以是否就可以基于PM提出一种XaaS呢。半虚拟化方面的感觉就是"bypass"，如果说VirtIO是数据通路的bypass, 那么SLO-Aware IO框架就可以说是功能性的bypass。这样通过bypass，我们就可以利用上PM设备的许多特性，如byte-addressing能力等。

应用实践方面, PM设备既不是纯存储又不是纯内存，所以当前的针对纯存储或存内存的算法需要改进该能利用上PM设备所以性能。因为PM设备的内存属性，我们可以利用它byte-addressing能力实现很多算法，并且需要经过实验验证。"新型存储设备上重复数据删除指纹查找优化"这篇论文讲的就是这方面的探索，通过实验分析得出结论，然后给出针对PM设备的算法改进意见。"An NVM SSD-based High Performance Query Processing Framework for Search Engines"这篇文章则提出了一种搜索框架来改进NVM SSD较DRAM速度慢的问题。

综上，给我感觉到的最终形态是: "我们的应用针对PM设备特性设计, 跑在专门根据PM优化的操作系统环境中，绕过操作系统绕过云平台, 最大程度的利用上所有性能"。


## 为什么对这个方向感兴趣

至于为什么对这个方向感兴趣，一方面是因为和我的知识框架比较契合, 我是从操作系统开始自底向上学习的, 之后依次学习了虚拟化, 分布式, 存储。所以就比较希望能从自己熟悉的领域出发，一点点接触新领域(搜索和算法)，从而不断扩展知识的边界。另一个方面，是出于未来的考虑, 我们做的设计应该是对未来的设计，就好比游戏厂商笃定摩尔定律生效会在现代开发次世代的游戏。结合大数据和云计算，我觉得这个方向就一个不错的选择。

在CMU15-445数据库系统课程学习中，我学习到了许多优化技巧，其中索引，缓存管理和执行计划方面的内容印象深刻。我们说只有应用知道自己需要什么，所以数据库开发者为了性能就会选择手动管理缓存，而不是让操作系统通用管理。还有出于对物理设备的特性做的考量，如磁盘顺序写性能会好，让读写尽量一次一批，缓存一次一页等。出于局部性原理可以适当选择预取策略等。执行计划和索引方面我们会考虑操作间，操作内的并行，考虑用什么算法处理当前数据分布模式，考虑用规约, 用布隆过滤器的方法减少不必要的操作等等。虚拟化方面我通过一个PowerPC虚拟化平台搭建的项目了解了虚拟化原理，深刻体会到如何通过软硬件结合解决系统优化问题，如基于EPT的地址转换，基于KVM的CPU执行等。感受到少一层抽象带来的性能提升，如VirtIO通过共享内存的方式减少数据拷贝和处理等。


# References

<!-- 1. Xiaoli Gong, Dingyuan Cao, Yuxuan Li, Ximing Liu, Yusen Li, Jin Zhang, Tao Li:A Thread Level SLO-Aware I/O Framework for Embedded Virtualization. IEEE Trans. Parallel Distributed Syst. 32(3): 500-513 (2021) -->
<!-- 2. Jiachen Zhang, Lixiao Cui, Peng Li, Xiaoguang Liu, Gang Wang, Towards Virtual Machine Image Management for Persistent Memory. MSST 2019:116-125. --> 
<!-- 3. 张佳辰,胡泽瑞,赵盛,施文杰,王刚,刘晓光, VMFS:一种持久性内存统一管理系统, 电子学报. 2021(12):2299-2306 -->
<!-- 4. Lixiao Cui, Kewen He, Yusen Li, Peng Li, Jiachen Zhang, Gang Wang,Xiaoguang Liu ,SwapKV: A Hotness Aware In-memory Key-Value Store for Hybrid Memory Systems, IEEE TKDE -->
<!-- 5. 何柯文，张佳辰，刘晓光，王刚，新型存储设备上重复数据删除指纹查找优化，计算机研究与发展. 2020,57(02)：269-280 --> 
<!-- 6. Xinyu Liu, Yu Pan, Yusen Li, Gang Wang, Xiaoguang Liu. An NVM SSD-based High Performance Query Processing Framework for Search Engines. IEEE Transactions on Knowledge and Data Engineering (TKDE), 2022 -->
<!-- 7. Yusen Li, Yunhua Deng, Wentong Cai, Xueyan Tang, Fairness-aware Update Schedules for Improving Consistency in Multi-server Distributed Virtual Environments, SimuTools 2016: 1-8 -->


[^1]: [张佳辰,胡泽瑞,赵盛,施文杰,王刚,刘晓光, VMFS:一种持久性内存统一管理系统, 电子学报. 2021(12):2299-2306]()
[^2]: [Jiachen Zhang, Lixiao Cui, Peng Li, Xiaoguang Liu, Gang Wang, Towards Virtual Machine Image Management for Persistent Memory. MSST 2019:116-125. ]()
[^3]: [Xiaoli Gong, Dingyuan Cao, Yuxuan Li, Ximing Liu, Yusen Li, Jin Zhang, Tao Li:A Thread Level SLO-Aware I/O Framework for Embedded Virtualization. IEEE Trans. Parallel Distributed Syst. 32(3): 500-513 (2021)]()
[^4]: [Lixiao Cui, Kewen He, Yusen Li, Peng Li, Jiachen Zhang, Gang Wang,Xiaoguang Liu ,SwapKV: A Hotness Aware In-memory Key-Value Store for Hybrid Memory Systems, IEEE TKDE]()
[^5]: [Xinyu Liu, Yu Pan, Yusen Li, Gang Wang, Xiaoguang Liu. An NVM SSD-based High Performance Query Processing Framework for Search Engines. IEEE Transactions on Knowledge and Data Engineering (TKDE), 2022]()
[^6]: [何柯文，张佳辰，刘晓光，王刚，新型存储设备上重复数据删除指纹查找优化，计算机研究与发展. 2020,57(02)：269-280 ]()
[^7]: [Yusen Li, Yunhua Deng, Wentong Cai, Xueyan Tang, Fairness-aware Update Schedules for Improving Consistency in Multi-server Distributed Virtual Environments, SimuTools 2016: 1-8]()

