---
title: Dockerfile语法
date: 2020-11-18
tags: docker
---

## 基本语法

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


##  构建镜像

- `docker build -t <image>[:version]`
    * `--build-arg`指定构造参数


## 其他

- STOPSIGNAL用于指定容器退出的信号
- HEALTHCHECK用于指定容器健康检查的配置
- SHELL用于指定脚本如windows镜像需要指定CMD /S /C
