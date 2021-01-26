---
title: Golang交叉编译
date: 2019-08-08 17:52:45
tags: 
- go
- tools
---


# <center>交叉编译<center>

所谓交叉编译就是在一个平台生成另一个平台的可执行文件。

- 通过`go tool dist list`查看支持情况
- 步骤：
    * 1. 设置目标平台以win-\>linux为例`set GOOS=linux`
    * 2. 设置目标的GPU`set GOARCH amd64`
    * 3. `go build`


## 实战

1.生产可执行文件	

```
// 指定目标环境
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build XXX
```



2.后台运行与关闭

```
// 生产可执行程序
nohup 程序路径 &  //后台运行,然后会在当前目录生成nohup.out问及文件，记录输出到内容

// 不生成可执行程序
nohup go run name &

//进程控制
ps –aux     //查看进程
ps –ef | grep name //查看name的进程
kill PID //关闭杀掉
```

2.开机自启

```

```

3.传文件

```
scp FILE NAME@IP:/root/
```

