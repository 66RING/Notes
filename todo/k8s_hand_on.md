---
title: title
author: 66RING
date: datetime
tags: 
- tags
mathjax: true
---

# Abstract

https://k8s.easydoc.net/docs/dRiQjyTY/28366845/6GiNOzyZ/puf7fjYr

# Preface

几种工具：

- `minikube`模拟器
- 云平台搭建, e.g. xxx云容器
- 裸机搭建
	* 主节点
		+ docker, 或者其他容器运行时
		+ kubectl，集群命令交互工具
		+ kubeadm，机器初始化工具
	* 工作节点
		+ docker, 或者其他容器运行时
		+ kubelet管理pod和容器，保证健康稳定运行
		+ kube-proxy网络代理

# Overview

## minikube基本使用

- `minikube start`启动集群
- `kubectl get node`查看节点
- `minikube delete --all`清空集群
- `minikube dashboard`可视化控制台服务

[more](https://minikube.sigs.k8s.io/docs/commands/)


## 裸机配置

- TODO
- 云主机实验
- 关闭selinux, 防火墙
- 安装程序
- 启动kubelet，docker
- 修改docker配置`/etc/docker/daemon.json`
	* 使用systemd作为cgroupdriver
	* 换源
- 主节点kubeadm初始化集群，安装控制台
	* 换源
	* 得到加入集群的命令
- 从节点加入集群
	* 使用`kubectl get node`查看情况


## 部署应用到集群

命令行或`yaml`配置文件传播。但是一个一个pod创建不方便，可以使用deployment管理

- `kubectl run ...`
- `kubectl apply -f <config.yaml>`
- `kubectl describe pod <pod name>`查看pod详情








