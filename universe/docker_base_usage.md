---
title: Docker常用命令
date: 2020-10-1
tags: docker
---

## Docker常用命令

- `docker ps [-a]`，查看正在运行的[所有]容器
- `docker <start|stop|rm> <ID|NAME>`，启动/停止/删除容器
    * `docker rmi <IMAGES>`，删除镜像
- `docker attach id`，进入某个容器，使用exit退出容器时，容器也会停止
- `docker exec -it <ID|NAME> </bin/bash>`，启动一个shell，以交互形式进入容器，exit时容器不停止
- `docker run --name NEWNAME -it CONTAINER [/bin/bash]`
    * 复制CONTAINER容器，并重命名为NEWNAME，[使用bash以交互模式进入]


## 基本操作

### 配置容器

- 1. 交互式编辑
    * `docker run -it <ID|NAME> </bin/bash>`，交互式进入，安装编辑器编辑对应的配置文件
        + 见`man docker <COMMAND>`
        + `-i`表示启动一个可以交互的容器
        + `-t`表示分配一个pseudo-tty
        + `-d`(detach)表示后台启动
    * 在一个TCP端口上绑定一个服务
        + `-p ip:[hostPort]:containerPort | [hostPort:]containerPort`
        + `-P`暴露容器所有端口到宿主机上的随机端口
    * 安装ssh服务后，使用ssh进入编辑
        + 1. `apt-get install openssh-server`
        + 2. 修改配置文件，"PermitRootLogin without-password"去掉注释免密登录
        + 3. `service ssh start`
- 2. `docker cp <CONTAINER:/path/to/conf> </source/file>`


### 管理容器数据

- 数据卷(volume)：
    * `-v[=[[HOST-DIR:]CONTAINER-DIR[:OPTIONS]]]`
    * TODO

