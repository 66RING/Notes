---
title: RDMA编程
author: 66RING
date: 2025-03-17
tags: 
- system
mathjax: true
---

# RDMA编程

- socket编程缺点
    * send/recv需要触发系统调用
    * 需要用户态到内核态数据拷贝
        + 用户空间物理地址不连续, 网卡无法直接访问
    * 主要需要CPU参与: 校验、封装、拷贝
- RDMA
    * RDMA网卡直接访问用户空间数据
    * RDMA自己处理校验、封装、拷贝

## abs

- WQ: 工作队列
    * WQE: 工作队列元素(任务)
- QP: Queue Pair
    * 两个WQ: send(SQ)/recv(RQ)
    * QPN: Queue Pair Number
    * e.g. 节点A的QP_id与节点B的QP_j通信
- CQ: Completion Queue, 完成的队列, aka **handle**
    * CQE: 完成队列元素
- WR: Work Request, 工作请求, aka **handle**
    * 用户发送的请求
- WC: Work Completion, 工作完成
    * 请求完成的信号
- MR: Memory Region
    * rdma通信中使用的内存, aka alloc
    * 功能
        + **用于虚拟地址到物理地址的转换**
            + 用户态权限有限，RDMA驱动要手动创建虚拟地址到物理地址的映射表
        + 控制访问权限, aka 只能访问允许/注册的区域
        + **避免换页后RDMA管理的虚拟地址和物理地址对不上**, aka 锁页, flag
- Protection Domain
    * 保护MR的访问权限不被破解
    * 机制: **QP对和MR绑定**, QP只能访问绑定的MR
- 主要流程
    * pre: 分配缓存、注册MR、交换元数据
    1. 用户通过wr向rdma驱动发送请求w/r
    2. 请求进入QP的的SQ/RQ
    3. 硬件将请求发送到网络, 将WC放入CQ返回给用户
    4. 用户通过驱动从CQ中取出WC, 完成请求
- 编程流程
    - client/server
        1. 申请空间: 注册PD，MR
        2. 拿到handle: 创建CQ(CPN)，QP(QPN)
        3. 交换元数据
        4. 调整QP状态
        5. RDMA通信:read/write
        6. 轮询CQ


## 环境模拟

> https://github.com/animeshtrivedi/blog/blob/master/post/2019-06-26-siw.md

## basic

> https://github.com/SYaoJun/rdma-example
>
> send/recv




































































