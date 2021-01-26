---
title: python hack
date: 2020-6-23
tags: 
- network
- python
- hack
---

## Scapy使用

需要root权限

scapy可以帮助我们铸造各种包，我们利用这些包可以实现我们想要的目的。如扫描等

- `sr`: Send ans Receive 

网络包有很多不同的参数，不需要死记硬背，只需用时看以下就行。如不知道ARP需要什么参数，可以采用如下，就可以知道

``` python
a = Ether()/ARP()
a.show()
```

不知道scapy的类有那些方法时，可以使用`type(arg)`来查看类，然后网上查找。


## 扫描探测

### ARP扫描/PING扫描

ARP协议完成IP地址和MAC地址的转换，询问目的机的MAC地址

通过ping扫描或arp扫描可以知道那些地址上有活动主机

``` python
pkt = IP(dst='47.96.226.37', ttl=1, id=168)/ICMP(id=188, seq=1)
result = srp(pkt)  # 得到 [收到的,未收到的]
result[0].res[x][y].getlayer(ARP).fields['key']  # res产生清单，很多元祖啊；getlayer是将二进制转换为字典，fields获取对应的值，缺省的获取整个字典
# 这个结构需要有所了解
```


### TCP扫描

利用tcp协议，扫描tcp的开放端口。发送SYN包，回复SYN ACK就说明端口开放 











