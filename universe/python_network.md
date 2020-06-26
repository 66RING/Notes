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
- 发送数据
    * udp: `s.sendto(data, (ip, port))`，如果没绑定端口则由系统随机分配
    * tcp: `s.send(data)`
- 接收数据
    * 1. 创建套接字
    * 2. 必须绑定端口，让程序使用固定的端口负责接收
    * 3. 接收`s.recvfrom(size)`，返回元祖

- 单工：只能收/发
- 办双工：能收发，要收就不能发，要发就不能收
- 全双工：能收发，如打电话

套接字可以同时收发，是全双工


### Socket中使用tcp

和udp的方法有些不同

- 建立连接
    * `s.connect((ip, port))`
- 发送数据
    * tcp: `s.send(data)`


### tcp服务器

一般需要以下步骤：
- 创建套接字
- bind绑定ip和port
- listen使套接字变为被动链接
    * 被动：即可以让别人来呼叫
- accept等待客户端的链接
    * 返回一个元祖，`(client_socket, client_addr)`
        + 可以通过`client_socket`向客户端发送数据
- recv/send收发数据





