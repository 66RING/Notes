---
title: docker多容器项目
date: 2020-11-22
tags: 
- operate
- docker
---

## docker容器交互

docker会通过一块虚拟的docker网卡为容器分配ip地址，即这些容器将在一个网段中，可以直接通过ip进行访问进行交互。但这里存在一个问题，我们需要手动登录一台机器查看ip，然后再另一台机器中访问，这在实际开发中是不可行的。

更方便的交互方式是使用`--link <name|id>[:<alias>]`，这样就可以通过,如`curl <alias>`自动解析域名。其本质就是在`hosts`文件中加入了`ip <alias> <id>`，把别名了容器id映射到了指定容器的ip。

看如下一个通信模型，只在nginx中暴露80端口，通过link的方式跟php通信，php又通过link的方式根mysql通信

```
nginx   link php expose 80
  |
  V
php     link mysql
  |
  V
mysql
```

**存在的问题** 通过link的通信时，容器的启动顺序很关键(容器都没启动，link到哪？)。所以在以后的重启/部署时指令输起来将比较麻烦

因此，可以将部署的指令记录到一个配置文件中，即使用`docker-compose`


## docker-compose

通过yml配置需要的服务，然后从yml中启动所有服务

```yaml
# docker-compose.yml

version: ""
services:
    serv1:
        image:
        links:
        volumes:
        and_some_other_docker_option:
    serv2:
        build: .  # 服务从dockerfile构建
        and_some_other_docker_option:
```

`docker-compose up`启动




