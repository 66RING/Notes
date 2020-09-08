---
title: RPC intro
date: 2020-9-3
tags: RPC
---

## RPC (Remote Procedure Call)

RPC，远程过程调用，是一个计算机通信协议。

相对的就是本地过程调用，即一个程序中调用它的子程序，可以直接通过地址访问。而RPC的远程，就是跨进程访问的意思。

该协议允许运行于一台计算机的程序调用另一个地址空间(通常是开放网络中的一台计算机)的子程序。程序员就像调用本地程序一样，无需额外地为这个交互作用编程。

RPC是一种Client/Server的进程间通信模式，程序分布在不同的地址空间里。

``` 
Client          Server
    |               ^
    v               |
proxy           proxy       动态代理层
    |               |
protocol        protocol    序列化/反序列化层
    |               | 
network   --->  network     网络传输层
```



