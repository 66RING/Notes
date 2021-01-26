---
title: CI/CD简介
date: 2020-11-22
tags: 
- operate
- CI/CD
---

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


### DaoCloud

#### 测试阶段

同样是关注远程仓库，然后执行一些自动化的脚本。DaoCloud的云端的基于一个docker镜像的，所以在配置测试环境时可以选择合适的docker镜像。


#### 构建阶段

在构建完成后会将镜像上传的daocloud的docker镜像仓库，方便以后通过`docker pull`部署到自己机器。当然可以在集群管理中配置主机，就可将主机绑定到daocloud平台，然后实现自动部署。


#### 发布阶段

添加部署到主机

*PS* :daocloud还提供了内网穿透，通过合理配置就可通过访问daocloud的域名访问(即只http)部署在内网的应用了



