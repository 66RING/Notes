---
title: Docker常用命令
date: 2020-10-1
tags: operate, docker
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
- 3. 保存镜像
    * `docker commit <ID|NAME> <NEW_IMG>`
        + 通过名字或id保存一个容器为镜像
    * 将镜像保存为`.tar`文件
        + `docker save <NAME> > <tar>`
        + 使用重定向符保存
    * 使用`load`从`.tar`文件加载镜像
        + `docker load < <.tar>`
        + 输入如保存对称


### 管理容器
#### 管理容器数据

使用数据卷(volume)，用于持久化共享数据

- `-v[=[[HOST-DIR:]CONTAINER-DIR[:OPTIONS]]]`
    * 将外部的目录/文件映射到docker内部
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


## Dockerfile

### 基本语法

- FROM
    * `FROM <image>`
    * 指定基于哪个镜像
- RUN
    * shell形式：`RUN <cmd>`
    * exec形式：`RUN ["exec", "arg1", "arg2"]`
    * 执行命令，**在docker build时运行**
- COPY
    * `COPY [--chown=<user>:<group>] <src>... <dst>`
        + 同样源和目的也可使用exec形式传递
    * 将文件或目录复制到容器中的指定目录
- CMD
    * 类似RUN，但运行的时间点不同
    * 在docker run时运行
    *  **指定启动容器时默认要运行的程序**
    * CMD指令可以被docker run中exec指定的程序覆盖
    * Dockerfile中有多个CMD，只有最后一个生效
- ENTRYPOINT
    * 类似CMD，但不会被dockr run中的命令行参数指定的指令覆盖，这些命令行参数会被当作参数传递给ENTRYPOINT指定的程序
        + 一般通过CMD传入变参，然ENTRYPOINT执行，如
            ```dockerfile
            ENTRYPOINT ["nginx", "-c"]  # 定参
            CMD ["/etc/nginx/nginx.conf"] # 变参  
            # 这么就实现了不传参执行默认参数，传参覆盖参数执行
            ```
        + 可以使用`--entrypoint`参数覆盖
    * Dockerfile中有多个ENTRYPOINT，只有最后一个生效
- ENV
    * `ENV <key> <value>`或`ENV <key1>=<value1> <key2>=<value2>`
    * 设置环境变量，以在后序指令中使用`$<key>`
- ARG
    * `ARG <arg>[=<default>]`
    * 构造参数，作用同ENV，但作用域不同
        + ARG仅在Dockerfile中有效(即build中)
        + 可以使用`docker build --build-arg`覆盖
- VOLUME
    * `VOLUME ["dir1", "dir2"...]`或`VOLUME <dir>`
    * 定义匿名数据卷。如果启动容器时忘记挂载数据卷，就挂载到匿名数据卷
        + 避免数据丢失
        + 避免容器不断变大
- EXPOSE
    * `EXPOSE <port1> [<port2>...]`
    * 声明(暴露)端口。运行使用随机端口映射时，即`docker run -P`，会自动随机映射EXPOSE的端口
- WORKDIR
    * `WORKDIR <dir>`
    * 指定运行时的主目录
- USER
    * `USER <user>[:<group>]`
    * 指定运行使用的用户和用户组
- ONBUILD
    * 当前镜像构建时不执行，当当前进行作为其他镜像构建时才执行


###  构建镜像

- `docker build -t <image>[:version]`
    * `--build-arg`指定构造参数


### 其他

- STOPSIGNAL用于指定容器退出的信号
- HEALTHCHECK用于指定容器健康检查的配置
- SHELL用于指定脚本如windows镜像需要指定CMD /S /C

