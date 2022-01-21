---
title: nginx中的负载均衡
author: 66RING
date: 2020-01-19
tags: 
- nginx
- distributed
mathjax: true
---

# Abstract

一个网页为了应对高并发的情景，常常会使用多台后台服务器还处理用户的响应，这种增加节点个数的扩展方式就称为"水平扩展"。而即使后台使用了成百上千台服务器，用户可以不需要跟每个服务器沟通的细节，不需要知道每个服务器的ip地址。这是正是因为在用户与服务器之间存在一个代理(proxy)，代理用户跟服务器通信。代理服务器中记录和保存了后台服务器的信息，怎么跟后台服务器通信、跟哪个后台服务器通信由代理节点说了算。代理节点将用户请求均匀和分发到各个服务器，这样就能避免请求集中到同一个服务器中造成服务器过载，这个过程就称为"负载均衡"。

基于成本和安全的考虑，后台服务器集群间通信常常使用内网，在外网的用户想要访问到处于内网的服务器需要经过代理服务器，因此代理服务器往往需要两个网卡分别负责外网通信和内网通信。而代理服务器转发从外网到内网的请求常常被称为"反向代理"。


# nginx

所谓代理服务，本质上也是通过软件自动化地监听端口然后转发，运行这种软件的节点就是代理服务器，或称为"负载均衡器"。市面上由很多提供代理服务的程序，其中nginx就是一种，这里将介绍如何配置安装nginx提供负载均衡服务。


## nginx安装

在linux环境中(ubuntu)，可以通过`apt install`命令安装nginx。

```
apt install nginx
```

可以通过`systemclt start nignx`命令启动nginx代理，也可以在命令行中输入`nginx`临时启动。默认情况下，nginx启动会会代理本机的80端口，通过浏览器访问`localhost:80`可以看到nginx的欢迎页面，说明nginx启动成功。

之后通过修改`/etc/nginx/nginx.conf`配置文件配置启动不同的负载均衡策略。


## nginx中的负载均衡策略

nginx支持多种负载均衡策略，用户可以根据具体场景配置使用不同的负载均衡策略。常见的有如下几种负载均衡策略。

1. 轮询
	- 简单地一个接一个轮流向服务器分发请求。一个一个轮流的方式，"公平"是符合直觉的
2. 固定权重
	- 根据一定概率(权重)向服务器分发请求。因为有可能有的服务器性能强悍，那他承担更多负担似乎也是合理的
3. IP哈希
	- 根据请求方的IP哈希值向服务器分发请求。好的哈希可以输入均匀地分发出去，那么考虑请求来自世界各地，对IP哈希就能均匀地将IP分配到各个服务器，也是一种公平的方案
4. 最短响应时间优先
	- 服务器的响应时间可以反应服务器的忙碌程度。和固定权重不同，权重需要运维人员手动计算和调整，是不太方便的，因此通过监听响应时间，也可以作为分配的依据


这里介绍几种方法

使用nginx的upstream配置可以配置不同的负载均衡策略。配置文件格式如下：

```
http {
	upstream myserver {
		...
	}

    server {
        ...
		location / {
			proxy_pass http://myserver;
        }

	}
}
```

括号最外层的`http`表示这个http请求相关的配置，内层的`server`就是http服务器相关的配置，`location / {}`就表示当访问服务器的`/`路径时将请求转发给`http://myserver`。而myserver又是一组`upstream`，通过配置upstream就可以配置转发时是以何种方式转发，也就是配置负载均衡策略。


### 轮询方式

先从最简单的轮询方式开始，可以在`upstream`中添加服务器url完成。

```
http {
	upstream myserver {
		server 127.0.0.1:3000;
		server 127.0.0.1:3001;
		server 127.0.0.1:3002;
		server 127.0.0.1:3003;
	}

    server {
        listen       80;
        server_name  localhost;
        location / {
			proxy_pass http://myserver;
        }
	}
}
```

这里添加了4个服务器，分别运行在`127.0.0.1`的`3000`到`3003`端口。轮询方式就将用户请求轮流发给这四个服务器。这几简单写个程序来测试一下。这里使用rust来编写服务器逻辑。

通过如下命令安装rust和cargo包管理工具

```
apt install cargo
```

然后使用`cargo new servers`创建服务器。在`servers/src/main.rs`文件中写入如下代码编写服务器逻辑：

```rust
use std::io::prelude::*;
use std::net::TcpListener;
use std::thread;

fn server(addr: &str, server_name: &str) {
    let listener = TcpListener::bind(addr).unwrap();
    println!("{} Running on on {}", server_name, addr);
    for stream in listener.incoming() {
        let mut stream = stream.unwrap();
        let mut read_buffer = [0; 200];
        stream.read(&mut read_buffer).unwrap();

        let msg = format!("Response from {}", server_name);
        let response = format!("HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}", msg.len(), msg);

        stream.write(response.as_bytes()).unwrap();
    }
}

fn main() {
    let handle = thread::spawn(|| { server("127.0.0.1:3000", "server0"); });
    thread::spawn(|| { server("127.0.0.1:3001", "server1"); });
    thread::spawn(|| { server("127.0.0.1:3002", "server2"); });
    thread::spawn(|| { server("127.0.0.1:3003", "server3"); });
    handle.join().unwrap();
}
```

- `main`函数创建4个服务器线程来模拟我们的服务器集群
- `server`函数用于创建服务器，响应服务器名
	* 首先通过`TcpListener`监听tcp连接
	* 对于所有tcp连接，返回Http响应，http响应格式是`response`遍历描述的内容的字节流
	* 最后通过`stream.write()`将http响应字节流返回

在`server`目录下使用`cargo run`命令启动服务

```
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/proxy_pass`
server0 Running on on 127.0.0.1:3000
server1 Running on on 127.0.0.1:3001
server3 Running on on 127.0.0.1:3003
server2 Running on on 127.0.0.1:3002
```

下面这个shell脚本配置curl进程http测试:

```sh
#!/bin/sh
# test.sh
declare -A servers
servers=(
  ["Response from server0"]=0 
  ["Response from server1"]=0 
  ["Response from server2"]=0 
  ["Response from server3"]=0
)

TOTAL=20

for ((i=0; i<$TOTAL; i++)) {
  resp=$(curl --silent localhost:80)
  echo $resp
  let servers["$resp"]++
}
echo "Server 0:" ${servers["Response from server0"]}
echo "Server 1:" ${servers["Response from server1"]}
echo "Server 2:" ${servers["Response from server2"]}
echo "Server 3:" ${servers["Response from server3"]}
```

- 这个脚本使用`curl`向nginx代理服务器发送请求
- 然后使用`echo`回显请求内容，并时相应计数器加一
- 最后`echo`打印各个服务器的响应情况

运行测试可以看到如下内容，

```
$ ./test.sh
Response from server0
Response from server1
Response from server2
Response from server3
...
Response from server0
Response from server1
Response from server2
Response from server3
Server 0: 5
Server 1: 5
Server 2: 5
Server 3: 5
```

可以看到，响应的会依次来自服务器0、1、2、3，然后再次循环。这就是轮询的含义


### 固定权值方式

更改配置文件如下

```
http {
	upstream myserver {
		server 127.0.0.1:3000 weight=1;
		server 127.0.0.1:3001 weight=3;
		server 127.0.0.1:3002 weight=2;
		server 127.0.0.1:3003 weight=4;
	}
}
```

然后使用`nginx -s reload`**重新加载配置文件**。再次使用上述`test.sh`脚本测试负载均衡策略，可以看到如下结果

```
$ ./test.sh
Response from server3
Response from server2
Response from server1
Response from server3
Response from server3
Response from server1
...
Response from server0
Response from server1
Response from server3
Response from server2
Response from server1
Response from server3
Server 0: 2
Server 1: 6
Server 2: 4
Server 3: 8
```

可以看到响应的服务器看起来是"随机"的，不该在最后的统计输出中可以看到各个服务器的响应次数，分别是`2/20`, `6/20`, `4/20`, `8/20`，正好对应配置中指定的权值，从而证明领导nginx的固定权值负载均衡的实现。


### IP哈希负载均衡方式

更改配置文件如下

```
http {
	upstream myserver {
		ip_hash;
		server 127.0.0.1:3000;
		server 127.0.0.1:3001;
		server 127.0.0.1:3002;
		server 127.0.0.1:3003;
	}
}
```

可以看到添加了`ip_hash`字段以启动IP哈希负载均衡。然后使用`nginx -s reload`**重新加载配置文件**。再次使用上述`test.sh`脚本测试负载均衡策略，可以看到如下结果

```
Response from server0
...
Response from server0
Response from server0
Server 0: 20
Server 1: 0
Server 2: 0
Server 3: 0
```

可以看到所有的响应都来自同一个服务器，因为请求都是从本机发出的，使用的是相同的IP，具有相同的哈希值，分配到相同的服务器。这是符号预期的。可以启动虚拟机来进行进一步的测试。虚拟机中输入如下命令：

```
curl <目标url>
这里的目标url是192.168.1.103

$ curl 192.168.1.103
Response from server2
$ curl 192.168.1.103
Response from server2
$ curl 192.168.1.103
Response from server2
```


可以看到，对于来自不同ip的请求分配给了另一个服务器处理，并对于这个IP都由该服务器处理。符合对IP哈希策略的预期

