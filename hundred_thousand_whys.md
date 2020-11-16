---
title: Hundred Thousand Whys
date: 2020-9-17
tags: tips, tools
---

## Why rust say it want to replace c++

Rust has a specfial  *Memory Management* , which is an **COMPILE-TIME memnory management base on Ownership, Borrowing and Lifetime** .

- C family: manually manage pointers and  **trust programmer access them and free correctly**. Otherwise wil cause run-time errors.
- Java: Run-time GC
- Rust: Ideally not to manage pointers manually, it has a compile-time borrow check.


## Diff in DOS, Mac and Unix file

The old Teletype machine used two character to satrt a new line. One to move the carriage(return \<CR\>) back to first position.

Back in the day, storage was expensive. Some people decided they did not need two character for end-of-line.

- The Unix people decided they use `Line Feed` only for end-of-line
- The Apple people decided they use `<CR>`
- The Window forks decided to keep the old `<CR><LF>`

## 什么是websocket

传统的http的客户端服务端模式是：客户端请求，等待服务端响应。因此服务端的相应很依赖于客户端的请求

socket的客户端服务端模式，一般通过tcp长连接，就可以实现服务端的主动相应。这在需要立即作出状态变更的场景是非常必要的，如网络聊天室、股票信息等。

传统的http一般通过轮循的方式来检测状态的变更，非常浪费带宽。因此，我们需要http，也需要socket，由此websocket就诞生了。


## 什么是REST接口规范

REST规范的http接口如下：

- `GET`，用于**查**操作
- `POST`，用于**增**操作
- `PUT`，用于**改**操作
- `DELETE`，用于**删**操作



