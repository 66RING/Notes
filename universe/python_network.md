---
title: python 网络编程
date: 2020-6-25
tags: python, network
---

## Socket基本使用

- 创建套接字对象`socket.socket(AddressFamily, Type)`
    * udp: `s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)`
    * tcp: `s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)`
    * 其中`AF_INET`表示IPv4，`SOCK_DGRAM`表示udp，`SOCK_STREAM`表示tcp
- 绑定端口`s.bind((ip, port))`让程序使用固定的端口负责接收
- 发送数据`s.sendto(data, (ip, port))`，如果没绑定端口则由系统随机分配
- 接收数据
    * 1. 创建套接字
    * 2. 必须绑定端口，让程序使用固定的端口负责接收
    * 3. 接收`s.recvfrom(size)`，返回元祖
