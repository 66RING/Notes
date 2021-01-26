---
title: iptables使用
date: 2020-11-23
tags:
- linux
- network
---

> 对网络上一些数据包通过表的形式进行限定或修改

## 三种表

- ~~Mangle~~
    * 一般在操作系统级别操作，这里讨论
- filter，过滤器
    * 对进出的数据包进行过滤
- nat，网络地址转换
    * 转换目的地址和目的端口，以及源地址源端口，做数据包转发
    * 如修改进入机器数据包的目的地址，使其转发到其他机器

iptables以链的形式组织配置规则, `iptables -t <table> -L`，就可以列出对应表中的规则

### filter表

查看一下filter表的内容

```
$ sudo iptables -t filter -L
Chain INPUT (policy ACCEPT)  # policy 表默认，这里全部接受
target     prot opt source               destination

Chain FORWARD (policy DROP)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

INPUT，OUTPUT，FORWARD就是filter表默认的三个链。进入的数据包就通过INPUT这个链来过滤，出去的数据包通过OUTPUT这个链来过滤。而FORWARD是在中间环节(路由)做的过滤，与nat表有关。


- 字段
    * target，表示规则类型(即使用的方法ACCEPT，DROP等)
    * prot，表示协议
    * opt，一些额外的选项
    * source，源地址端口
    * destination，目的地址端口
- 设置规则`iptables [OPTIONS...]`
    * `-t <table>`，设置`table`表
    * `-A <Chain>`，将规则append到指定链上，插在末尾
    * `-I <Chain> rulenum`，将规则insert到指定链上，插在第rulenum条，默认1
    * `-D <Chain> rulenum`，删除链上第rulenum条规则
    * `-j <target>`，指定规则的类型，ACCEPT，DROP等
    * `-p <protocol>`，过滤的协议
        + `--dport <dest_port>`，过滤的目的端口
    * `-d <dest>`，过滤的目的地址
    * 例：`iptables -t filter -A INPUT -j DROP -p tcp --dport 8081`
        + 如果收到一个包，这个包的目的端口是8081，用的tcp协议，则丢弃


### nat表

nat表包含如下内容

```
$ sudo iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
```

当本机作为中间人做端口转发时，数据包经过如下路径1。如果是通过本机直接发出或接收数据包，即不是做转发，则不会经过`PREROUTING`和`POSTROUTING`的完整流程，修改源地址和目的地址需要使用nat表的`OUTPUT`和`INPUT`链

```
1. packet in --> PREROUTING(改目的) --> FORWARD --> POSTROUTING(改源) --> packet out
2. packet in --> INPUT --> APP
3. APP --> OUTPUT --> packet out
```

PREROUTING表示路由之前，用于改目的地址;POSTROUTING表示路由之后，用于改源。

- 设置规则`iptables [OPTIONS...]`
    * `-j DNAT`，对目的进行转换
        + `--to <ip:port>`，转换到ip
    + `-j SNAT`，对源进行转换
        + `--to <ip:port>`


### 实验

要求: 把192.168.0.12:7788端口的数据包转发到192.168.0.11:7799。如在192.168.0.11:7799开了一个nginx服务，现要在192.168.0.12:7788中访问

- 服务器1
    * 外网ip：123.121.0.3
    * 内网ip：192.168.0.11
- 服务器2：
    * 内网ip：192.168.0.12
        + 7788端口提供服务
- **要求**：
    * 设使用ip为1.2.3.4:5的机器访问
    * 1. 通过外网访问服务器1的7799端口来获取服务器2中7788端口的服务
    * 2. 服务器1本地访问192.168.0.11:7799端口来访问服务器2中7788端口的服务

*PS*，请求是发送的，别忘了处理响应。


#### 要求1

```sh
# 当前机器：服务器1

iptables -t nat -A PREROUTING -p tcp -d 123.121.0.3 --dport 7799 -j DNAT --to 192.168.0.12:7788
# 当请求服务器1 7799端口时，转发到服务器2的7788端口

iptables -t nat -A POSTROUTING -p tcp --d 192.168.0.12 7799 -j SNAT --to 192.168.0.11

iptables -t filter -A FORWARD -j ACCPET
```

| 时期             | 操作         | 源ip:port     | 目的ip:port       |
|------------------|--------------|---------------|-------------------|
| packet in        | 访问         | 1.2.3.4:5     | 123.121.0.3:7799  |
| PREROUTING       | DNAT         | 1.2.3.4:5     | 192.168.0.12:7788 |
| routing decision | 判断是否转发 | 1.2.3.4:5     | 192.168.0.12:7788 |
| POSTROUTING      | SNAT         | 123.121.0.3:X | 192.168.0.12:7788 |
| packet out       | 转发         | 123.121.0.3:X | 192.168.0.12:7788 |


#### 要求2

```sh
# 当前机器：服务器1

iptables -t nat -A OUTPUT -p tcp -d 192.168.0.11 --dport 7799 -j DNAT --to 192.168.0.12:7788
iptables -t nat -A OUTPUT -p tcp -d 123.121.0.3 --dport 7799 -j DNAT --to 192.168.0.12:7788

# 当请求服务器1 7799端口时，转发到服务器2的7788端口
# 因为是本地访问，不会经过PREROUTING的过程，因此只能通过OUTPUT链转发

iptables -t filter -A FORWARD -j ACCPET
```

| 时期       | 操作 | 源ip:port      | 目的ip:port       |
|------------|------|----------------|-------------------|
| packet in  | 访问 | 192.168.0.11:X | 192.168.0.11:7799 |
| OUTPUT     | DNAT | 192.168.0.11:X | 192.168.0.12:7788 |
| packet out | 转发 | 192.168.0.11:X | 192.168.0.12:7788 |


#### 开启内核转发

去掉注释`/etc/sysctl.conf`，使生效`sudo sysctl -p`

```sh
# net.ipv4.ipv4_forward=1
```

或临时修改

`echo "1" > /proc/sys/net/ipv4/ip_forward`


