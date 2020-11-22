---
title: RPC intro
date: 2020-9-3
tags: RPC, tech
---

## RPC (Remote Procedure Call)

RPC，远程过程调用，是一个计算机通信协议。

相对的就是本地过程调用，即一个程序中调用它的子程序，可以直接通过地址访问。而RPC的远程，就是跨进程访问的意思。

该协议允许运行于一台计算机的程序调用另一个地址空间(通常是开放网络中的一台计算机)的子程序。程序员就像调用本地程序一样，无需额外地为这个交互作用编程。

RPC是一种Client/Server的进程间通信模式，程序分布在不同的地址空间里。

``` 
Client          Server
    |               ^
    v               |
proxy           proxy       动态代理层
    |               |
protocol        protocol    序列化/反序列化层
    |               | 
network   --->  network     网络传输层
```



## Go语言RPC实例

### 入门

GO语言的RPC包路径为`net/rpc`


- 一个对象要能走RPC调用需要规则
    * Go语言的RPC规则：公开方法只能有两个可序列化的参数，第二个参数为指针类型，并返回一个error
- 注册RPC服务`rpc.RegisterName("NameSpace", new(对象))`
    * 会将对象类型中所有满足RPC规则的对象方法注册为RPC函数
    * 所有注册的方法会放在"NameSpace"的命名空间下
- 拨号RPC服务`client, err := rpc.Dial("tcp", "localhost:1234")`
    * 这里建立通过tcp调用本机1234端口的rpc服务的连接
- 调用RPC服务`client.Call("NameSpace.Func", "arg1", &args)`
    * 第一个参数用点连接RPC服务的名字和要调用的方法
    * 第二、三个参数，是RPC需要的两个参数
-  **安全的RPC调用** 
    * 上面就实现的简单的RPC调用过程，但是在**RPC方法名，参数类型等还存在安全隐患**。所以更安全的RPC调用需要进行一些简单的封装
        + 如：
            ```go
            func (p *RPCServerClien) SomeFunc(request string, reply *string) error{
                return p.Client.Call("NameSpace.SomeFunc", request, reply)
            }
            ```



