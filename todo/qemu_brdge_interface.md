---
title: linux虚拟网桥创建
author: 66RING
date: 2022-01-15
tags: 
- tool
- linux
mathjax: true
---

- ethernet
	* https://wiki.archlinux.org/title/Network_configuration/Ethernet
- https://www.youtube.com/watch?v=rSxK_08LSZw&ab_channel=SethJennings
- https://wiki.archlinux.org/title/Network_bridge
- https://apiraino.github.io/qemu-bridge-networking/
- https://wiki.artixlinux.org/runit/NetworkConfiguration

arch

创建网桥`br1`

```
ip link add name br1 type bridge
```

启动网桥`br1`

```
ip link set dev br1 up
```

接入网桥的网卡

```
ip link set dev <网卡名> master <网桥名>
```

移除

```
ip link set dev <网卡名> nomaster <网桥名>
```

删除网桥

```
ip link delete dev <网桥名>
```

## 2

qemu中使用网桥

```
-net nic -net bridge, br=<网桥名>
```


# Abstract


# Preface


# Overview

# samba usage

testparm 测试

smbstatus

/etc/samba/smb.conf

# route

- https://linuxconfig.org/configuring-virtual-network-interfaces-in-linux

(host)virtual interface --- bridge ---- (guest) interface
