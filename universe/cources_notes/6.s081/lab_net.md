---
title: MIT6.S081 network lab
author: 66RING
date: 2022-01-01
tags: 
- OS
mathjax: true
---


# networking

编写一个xv6设备驱动程序，驱动网卡

## Background

> 参考书Chapter5 中断和设备驱动

你会使用一个叫作e1000的网络设备来处理网络通信。qemu虚拟化e1000，e1000就相当于现实中真正链接网络的网卡设备。

xv6的ip地址为10.0.2.15。qemu会让宿主机(the computer running qemu)的ip地址为10.0.2.2。qemu会将网络数据包传递给宿主机。


你将使用qemu的"user-mode network stack"。具体可查看qemu的文档。makefile已经起用了e1000网卡和"user-mode network stack"。

makefile配置qemu，让他记录所有网络包输入输出到`packets.pcap`文件中。可以通过查看此文件来调试。

```
tcpdump -XXnr packets.pcap
```

`kernel/e1000.c`包含了e1000的初始化代码和传送网络包的空函数，你将要实现这些空函数。`kernel/e1000_dev.h`定义了e1000的寄存器和标志位，具体根据[e1000手册](https://pdos.csail.mit.edu/6.S081/2020/readings/8254x_GBe_SDM.pdf)。`kernel/net.c`和`kernel/net.h`包含了实现了IP, UDP和ARP的简单的网络协议栈。这些文件同样使用固定大小的数据结构来保存网络包(`mbuf`)。最后`kernel/pci.c`包含xv6启动时在pci总线上寻找e1000设备的代码。


## Your Job

完成`kernel/e1000.c`中的`e1000_transmit()`和`e1000_recv()`让驱动可以传输网络包

具体问题可以查看[e1000手册](https://pdos.csail.mit.edu/6.S081/2020/readings/8254x_GBe_SDM.pdf)。下面一些章节可能会比较有用

- Section2 是必须的，会给overview
- Section3.2 包接收
- Section3.3和Section3.4 包传输
- Section13 e1000使用的寄存器
- Section14 帮助理解初始化代码

阅读e1000手册，你需要熟悉Chapter 3, 4.1和14(可以先跳过2)。还会需要参考Chapter13。不需要关注太多细节，只需要实现基础功能。

`e1000.c:e1000_init()`函数配置e1000设备从**ram中读写**网络包。因为用了DMA。

因为数据包的达到速度可能会大于处理速度，`e1000_init()`供多个缓存让e1000设备使用。

> **The E1000 requires these buffers to be described by an array of "descriptors" in RAM**; each descriptor contains an address in RAM where the E1000 can write a received packet.

e1000将缓冲区组织成"descriptor数组"，e1000将向descriptor中的地址收发数据。

`struct rx_desc`描述了descriptor的格式，descriptor数组是环形组织的(循环数组)。`e1000_init()`会创建`mbuf`数据包缓冲区。另外还有一个传输环(transmit ring)，驱动将想要e1000发送的数据包放入。`e1000_init()`配置这两个环的大小分别文`RX_RING_SIZE`, `TX_RING_SIZE`

- 接收环
- 发送环

网络协议栈位于`net.c`，发包使用`e1000_transmit()`和`mbuf`中的数据包。你的实现必须使用一个数据包指针，指向传输环(transmit ring)的descriptor中的数据。`struct tx_desc`定义了结构。你需要保证每个`mbuf`当e1000完成数据包传输后(e1000会将descriptor的`E1000_TXD_STAT_DD`位置位)最终会被释放。

- 传输完成`E1000_TXD_STAT_DD`置位

当e1000接收到网络数据包，它首先会用DMA将数据包传输到`mbuf`的"next RX(receive) ring descriptor"，然后产生中断。你的`e1000_recv()`实现需要扫描RX ring然后将`mbuf`中新的数据包使用`net.c:net_rx()`发送给网络协议栈。然后你需要申请新的`mbuf`然后放入descriptor，为下次dma做准备。

- `net.c:net_rx()`发数据包给协议栈

除了要读写descriptor ring外，你的驱动还需要通过e1000映射到内存的控制器与e1000交互。以便检测是否由新数据包和通知e1000发送。全局变量`regs`保存了指向e1000第一个控制器的指针(数组)，你的驱动可以索引该数组访问到其他控制器。你将使用`E1000_RDT`和`E1000_TDT`

- 与内存中的控制器交互
	* `E1000_RDT`, `E1000_TDT`

测试: 一个窗口中`make server`，另一个窗口`make qemu`和`$ nettests`。`nettests`的第一个测试会尝试发送UDP数据包给宿主机。如果实验完成，`make server`会发送响应。


## hints

在`e1000_transmit()`和`e1000_recv()`中添加打印

实现`e1000_transmit`

TODO整理整个流程

- 读取`E1000_TDT`寄存器，向e1000寻找它希望的next packet的TX ring索引
- 检查ring是否溢出。如果`E1000_TDT`索引的descriptor status的`E1000_TXD_STAT_DD`没有置位，说明e1000没有完成响应传输请求，返回错误
- 否则，使用`mbuffree()`释放从descriptor传输来的最后一个`mbuf`(如果存在的话)
- 之后填充descriptor
	* `m->head`指向内存中的数据包
	* `m->len`表示数据包的长度
	* 设置必要的cmd标志位(查看手册Section 3.3)
	* 然后stash away a pointer to the mbuf for later freeing。保存一个`mbuf`指针，方便之后释放
- 最后更新ring的位置使用`(1 + E1000_TDT) % TX_RING_SIZE`
- 如果`e1000_transmit()`成功向ring添加`mbuf`则返回0。失败返回-1，然后调用者释放`mbuf`

实现`e1000_recv`的提示

- 找到ring中待接收的包(the next waiting received)的位置：使用`(E1000_RDT + 1)%TX_RING_SIZE`, 注意是加1取模
- 查看对应descriptor的状态: 检测`status`成员的`E1000_RXD_STAT_DD`位检查新数据包是否就绪，如果没就绪则停止
- 否则更新`mbuf`的`m->len`为descriptor的记录的长度。使用`net_rx()`传递`mbuf`给网络协议栈处理
- 之后使用`mbufalloc()`申请新的`mbuf`来替换刚刚传递给`net_rx()`的旧`mbuf`
	* 初始化新`mbuf`: 将新`mbuf`的数据(`m->head`)指向descriptor记录的的`addr`，然后清除descriptor的状态位为0
- 最后更新`E1000_RDT`寄存器为ring中最后处理的descriptor
- `e1000_init()`使用`mbuf`初始化RX ring，你可以参考它是怎么做的
- 会出现到达的包的数量超过ring的大小的情况，需要注意处理

因为xv6会存在多个线程使用e1000的情况，需要注意加锁。或者也可以在内核线程中处理使用e1000并处理中断。


## result

机制小论文

- 需要熟悉基本框架
- 了解DMA工作
	* 描述符(`descriptor`)和具体接收数据的地方
- 这个实验需要了解的东西比较多，但是跟着hint做就行
- mbuf index and descriptor index
- ring buffer的巧妙地方
	* 如`transmit`中用完当前mbuf后，指向下一个mbuf。当获取的mbuf非空时，说明就是上一次使用的`mbuf`，这时应该页过了一段时间了，缓存也够久了，再释放就能用了
	* 即头遇到尾了就释放过期数据。`E1000_TDT` TX Descripotr Tail
- 机制
	* 使用ring buffer
	* 设备的缓冲区, 如果`tx_mbufs`，设备接收到的数据放入其中
	* descriptor和mbuf一一对应，使用同样的index，难道设计就是"descriptor of mbuf?"
		+ 即`desc`对应全局的`tx_mbufs`和`rx_mbufs`怎么用
- data are stored in buffers pointed by descriptor
- DNS测试可能需要科学上网，不过可以修改`nettests`的`dst`


