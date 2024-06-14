---
title: 使用container名再容器之间互ping
author: 66RING
date: 2022-03-19
tags: 
- docker
- misc
mathjax: true
---

# 使用container名再容器之间互ping

创建两个容器:

```bash
docker run -it --name node0 ubuntu
docker run -it --name node1 ubuntu
```

使用`docker inspect`查看node0的ip

```
docker inspect node0 | grep IPAddr
```

容器内按照ping攻击`apt install iputils-ping`

使用ip做ping可以发现ping成功

使用container名字ping可以修改host文件, 也可以在主机端操作。

1. 使用`docker network create net1`创建一个网络
2. 使用`docker network connect net1 node0`, `docker network connect net1 node1`将两个container加入网络中

之后可以使用container名字做ping







