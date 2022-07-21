---
title: XPC - Architectural Support for Secure and Efficient Cross Process Call
author: 66RING
date: 2022-07-12
tags: 
- papers
mathjax: true
---

# XPC: Architectural Support for Secure and Efficient Cross Process Call

### abs

- 数据和逻辑分离, matedata和data分离
	* 还有其他?
	* 内核值参与简单的工作 -> 解决传输的问题

- XPC如何解决 TOCTTOU问题?


- xcall, xret
- xcall-entry
- xcall-cap
- relay-seg
	* addr mapping machanism -> ownership -> 防止TOCTTOU 且不会TLB shootdown
	* relay-seg传递, "handover"


### 动机

- 动机
	* 实验发现大量使用用在的transform和switch
		+ 数据量大时大量的数据传输
		+ 数据量小时CPU大量消耗在switch

- 解构IPC, **很cool的研究方法**
	* **基于state-of-art seL4的实现**
		+ fast path: 不可中断
		+ slow path: 可中断
	* trap and store(切换)
		+ 切换不可避免的原因是callee和caller之间无法相互信任
			+ 但是一些情况下又是相互信任的, 如calling convention. 怎么传参
			+ 因此我们是否可以做更多约定来保证信任
	* IPC logic(检查)
		+ 主要就是访问控制
			+ **可以多用用硬件**
			+ **逻辑和控制分离**
				+ what if, metadata和data分离
		+ xcall-cap
	* process switch(调用)
		+ 内核调度出队入队call thread
			+ callee回复 -> 用`reply_cap`
			+ msg传输
			+ switch to callee thread 
		+ 整个过程极大影响缓存，从而影响IPC性能
	* message transfer(通信)
		+ IPC的主要开销
		+ 三种方法(多阶段处理方案)
			1. 小数据直接寄存器
			2. 中数据 -> slow path -> 拷贝数据
			3. 大数据 -> fast path -> 用户态共享内存
		+ 共享内存能实现零拷贝, 但是很难兼顾安全
	* 结论
		+ 不用引入内核
		+ 尽量零拷贝


### 设计

> "冯诺依曼结构是否过时?"
>
> 是否可以参考事件驱动模型
>
> 分形相似中断处理过程

- User-level Cross Process Call
	* xcall-table, 保存xcall-entry的映射关系
		+ 保存在新寄存器中`-reg`
	* xcall-entry, 类似中断号, 对应处理程序
	* xcall-cap, 鉴权
	* `xcall-cap` + `xcall #reg(id)`
	* 主要思路就是硬件保护
- Lightweight Message Transfer
	* `seg-reg`寄存器指向一段**连续物理地址**: `relay-seg`
		+ `relay-seg`
			+ 为何要连续?? cool
				+ 因为`relay-seg`中的内容要**绕过内核: 不使用页表，直接根据寄存器转换**
		+ 又回到了解构IPC中提到的calling convention: 如何传递参数
		+ 内保证`relay-seg`区域不会影响不重叠到其他地址空间
			+ 从而也不会TLB shootdown(本质就是缓存一致性导致的)
			+ 不重叠不也是另一方面的无共享嘛??
- XPC Programming Model(似乎又分形相似了RPC了呢)
	1. 服务注册: `x-entry`(handler)
	2. "服务发现": 用户获取服务id与cap
	3. 服务调用: `xcall #reg(x-entry ID)`, 通过共享内存信息
		- client是如何将msg放入relag-seg区域的??如何准备时就绕过页表的??使用虚拟内存中的直接映射机制??
			* `alloc_relay_mem(size)`
			* `xpc_call(server_ID, xpc_arg)`


### XPC Engine

- `xcall`: **XPC Engine's four task**
	* check cap, at `#reg`
	* load target x-entry from table
	* linkage record?? push to likage stack
	* 加载新页表, 设置PC寄存器
- `xret`
	* 将linkage record弹出
	* 返回原进程
- `xcall-cap`
	* bitmap表示
	* 存储在thread自己的内存区域中
	* 由`xcall-cap-reg`寄存器指向
	* 由内核维护
- **link stack** ??
	* 记录调用信息, 仅内核能访问
		+ 页表指针, 返回地址, xcall-cap-reg等 
		+ "一个新的, 有权限控制的状态空间, 使用'新的栈'"
	* "懒保存, 异步", save linkage record lazily
- XPC Engine Cache
	* 用于优化x-entry和cap的加载
	* 两个考虑:
		1. 局部性
		2. **可预测**: 如调用者知道自己要到啥


- "银弹"
	* 引入硬件, 内核态预处理



### Relay Segment

- relay-seg
	* `seg-reg`寄存器
		+ 比页表优先级高
		+ 记录4个域: 物理地址base, 虚拟地址base, len, perm
- seg-mask
	* relay-seg用户虽然不可用直接访问, 但是可以通过seg-mask缩小范围, 然后哦再传递给其他被调用者
	* xcall时, seg-reg的范围就会根据mask变化
- Multiple relay-seg
	* 内核管理的per-thread memory seg-list
	* 通过`swapsg #reg`切换使用不同relay-seg
- **Ownership of relay-seg**
	* 对抗TOCTTOU攻击
	* 一个relay-seg一次只会被一个core激活使用
		+ 类似"atomic"??
- Return a relay-seg
	* 根据"栈"(linkage record)中的信息恢复


## TODO

- 优化点?
	* XPC选择同步方式 -> 适应现存系统
- 所有`??`标记
- 所有cool标记
- scalability问题和虚拟化场景下IPI(Inter-Processor Interrupt)问题
- 我们对计算机的抽象是否过时?? 分布式系统, 服务注册，自治系统??

### Sum

- 引入硬件, 安全地在特权级下"预处理"
	* 预处理指: 只做一些必要的工作, 对后续工作提供安全保证
- "内核就该做管理", 这样很多任务就能offload出内核了
- "引入新的地址空间, 用linkage recore做该空间的'栈', 以保存信息"
- 基于owerership的一种协作式机制
	* 核心思路就是没有所有权时yeild掉
	* 但怎么防止死锁呢??
- IPC, 可预测, 直接预取
- 多级预案: 小数据, 中数据, 大数据各自方案

- radix-tree优化bitmap的可扩展性问题


### QA

- 像数据库这种极度依赖自己做内存管理的系统, IPC微内核是否也能做出很好的抽象??
	* 虽然可以外核架构
- 既然我们引入了新硬件, 那么在云计算时代, 有VMM层的加持, 我们是否可以进一步引入各种"virtual-reg/device"等概念来辅助完成呢??
- 是否可以存在一种通用硬件抽象, 如某种"customize table"用于提供额外的寄存器列表。然后内容是只能由内核进行修改, 用户态只读, 这样XPC中xcall-cap, relay-seg等就有了通用的解决方案



