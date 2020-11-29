---
title: 配置虚拟机集群及ssh
date: 2020-3-27
tags: operate, linux
---

## 配置虚拟机集群

我也许没有很多太服务器，但是我可以在本地将他们虚拟出来鸭!只需要几步就能搭建起自己的多节点环境了

- 创建几个虚拟机
- 配置虚拟机网络

创建虚拟机就不多介绍了，直接跳到配置虚拟机网络部分。这里以`centos`为例

- 配置虚拟机网络位桥接模式
    * [原因](https://www.junmajinlong.com/virtual/network/vmware_net/)
    * [原因](https://www.junmajinlong.com/virtual/network/virtualbox_net/)

联网后用`dhclient`获取一个ip。**并修改网络配置文件**。如我为网卡`ens0p8`分配了`192.168.123.105`，然后修改对应的配置文件(如果网卡没有对应的配置文件可以拷贝别的网卡配置文件然后进行修改)

```sh
cat /etc/sysconfig/network-scripts/ifcfg-ens0p8
```

会得到如下:

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="689baf40-a22b-4e2b-b343-14fcbfe651f4"
DEVICE="enp0s3"
ONBOOT="yes"
```

我要将机器作为节点使用所以需要固定ip地址和开机自动启动，所以要令`BOOTPROTO=static`和`ONBOOT="yes"`，然后写下固定的ip`IPADDR="192.168.123.105"`，此外还需要设置子网掩码，网关，DNS等

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s8"
UUID="689baf40-a22b-4e2b-b343-14fcbfe651f4"
DEVICE="enp0s8"
ONBOOT="yes"
IPADDR="192.168.123.105"
GATEWAY="192.168.123.1"
NETMASK="255.255.255.0"
DNS1="8.8.8.8"
```

保存退出，然后重启网络服务，一个节点就这么设置完毕，其他节点同理


## 配置ssh

默认防火墙的开启的，所有要为ssh开个门

`firewall-cmd --zone=public --add-port=22/tcp --permenent`

`--permenent`表示永久生效
