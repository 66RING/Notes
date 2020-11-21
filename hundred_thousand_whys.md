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


## 何时发生用户态和内核态的切换？

- 用户态到内核态
    * 申请外部资源时
        + 硬件
        + 系统调用
        + 中断
        + 异常
- 内核态到用户态
    * 需要在内核态处理的事完成时


## 什么是CI/CD

CI/CD：持续集成和持续交互。代码提交到代码仓库后自动触发一些自动化的流程。CI/CD的工具就是干这用的。

- 什么是DevOps
    * DevOps是一种思想方法论，涵盖开发、测试、运维的整个过程。强调通过自动化的方法管理软件变更，软件集成

```
plan    --> code    --> build  --> test         Dev
  ^                                 |
  |                                 V
monitor <-- operate <-- deploy <-- release      Ops
```


### Jenkins

- 原理：
    * 代码提交到远程仓库
    * 如果检查到远程仓库有更新，则拉到本地然后执行构建脚本
- 在本机运行，所以需要一个自己的机器
 

### TravisCI

TravisCI是github的一个合作伙伴，提供云端机器运行自动化脚本，通过TravisCI官网就能开启github项目的CI服务，之后要在项目中添加`.travis.yml`文件

*PS* : github自带类似的工具，github action

基本用法：

```yml
# .travis.yml
language: lang  # 写执行脚本前要先指定语言

before_script:
    - some scirpt

script:
    - some scirpt
```


