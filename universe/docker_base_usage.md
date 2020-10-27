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
- `docker inspect <CONTAINER>`查看详情


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
        + `-P`暴露容器所有端口到宿主机上的一个随机端口，如`0.0.0.0:32768->33060`
    * 安装ssh服务后，使用ssh进入编辑
        + 1. `apt-get install openssh-server`
        + 2. 修改配置文件，"PermitRootLogin without-password"去掉注释免密登录
        + 3. `service ssh start`
- 2. `docker cp <CONTAINER:/path/to/conf> </source/file>`


### 管理容器
#### 管理容器数据

使用数据卷(volume)，用于持久化共享数据

- `-v[=[[HOST-DIR:]CONTAINER-DIR[:OPTIONS]]]`
    * 可以挂载多个数据卷
    * 挂载主机目录或文件作为数据卷的话必须使用绝对路径
    * OPTION可以设置权限，ro只读
- 创建和挂载数据卷容器：有一些数据可能需要在容器间共享，故创建数据卷容器是个不错的选择
    * 使用`--volumes-from <CONTAINER>`来挂载其他容器中的数据卷
        + 可以挂载多个，只要是挂载了数据卷的容器就行
- 删除容器数据卷并不会删除。删除最后一个还挂载它的容器时使用`docker rm -v`来同时删除


#### 管理容器通信

- 使用端口映射的方式通信
    * 顾名思义
- 使用link方式通信，
    * links允许容器发现另一个容器，并在期间建立一个安全的通信以便交换数据， **不会暴露任何端口** 
    * `--link <name|id>[:alias]`
    * 一个连接允许源容器提供信息给目标容器，如数据库

##### Link的原理

环境变量
