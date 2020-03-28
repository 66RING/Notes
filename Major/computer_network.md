---
title:  自顶向下计算机网络
date: 2020-3-25
---

#  自顶向下计算机网络

## 基本概念

- ISP: Internet Server Provider
- 拓扑：连线的方式
- 吞吐量：水管的大小
- 流量：网速
- 网络大小划分  
	- PAN:personal area network
	- LAN:局域网
	- MAN:城局域网
	- WAN:广局域网
	- 互联网（最大）


## 参考模型  

### 分层原理

信宿机第n层收到的对象应与信源机第n层发出的对象完全一致.
好比做飞机的登机口和下机口

每一层都为他的上一层服务  
信息发送方要做什么:  
- 封装/打包:  
	- 然后从最高层逐层传到物理层
- 每一层上，数据都会被加上头部信息，用于信息传递  

收方要做什么:  
- 解封装/解包  


#### 典型的分层模型:

- ISO OSI 七层模型  
	- 应用层
    - 表示层
    - 会话层
    - 传输层
    - 网络层
    - 数据链路层
    - 物理层
- TCP/IP (DoD) 四层模型  
	- 应用层
    - 传输层
    - 网络互联层
    - 主机到网络层


## 应用层

### 应用程序体系结构

- 客户-服务器体系结构
    - 至少一个服务器主机负责处理多个来自客户主机的请求
- P2P体系结构
    - 主机到主机, 这些主机称为对等方

### 基本概念

- 套接字: socket
    - 进程通过socket的软件接口向网络发送报文和传输报文
        - 好比房子的大门
    - 套接字地址: ip + port
- API: Application Programming Interface

### HTTP

HTTP(HyperText Transfer Protocal, 超文本传输协议)是Web的核心, 客户端和服务器通过交换HTTP报文进行会话. HTTP使用TCP作为它的支撑传输协议, 即先建立TCP连接, 再通过TCP连接向彼此套接字发送报文. 

- TCP为HTTP提供了可靠数据传输服务
    - 即发出的每个请求到能完整的到达目的地
- HTTP是一个无状态协议


- 持续连接的HTTP
    - 可以一个接一个发送请求
- 非持续连接的HTTP
    - 发送一个对象后TCP连接关闭


#### HTTP报文格式

请求报文
- 请求行
    - 方法 URL 版本
    - 包含一系列必要信息
- 首部行
    - 首部字段: 值
    - 相当于指定配置
- 空行
- 实体主体
    - 使用POST方法时才使用

``` 
--- 请求行 ---
GET /some/dir HTTP/1.1
--- 首部行 ---
Host: www.some.com           # 指明对象所在的主机, 该首部行的Web高速缓存所要求的
Connection: close            # 非持续连接
User-agent: Mozilla/5.0      # 发送请求的浏览器类型
# 还会有很多的配置
```

HTTP使用的方法
- GET
- POST
- HEAD
    - 类似GET, 但不返回请求对象
- PUT
    - 允许用户上传对象到指定Web服务器
- DELETE
    - 允许用户删除服务器上的对象

响应报文
- 状态行
    - 版本 状态码 短语
    - 常见状态码
        - 200 请求成功
        - 301 请求的对象已被永久转移
        - 400 Bad Request 请求不能被理解
        - 404 Not Found 请求的文档不在服务器上
        - 505 服务器不支持请求报文的HTTP版本
- 首部行
- 空行
- 实体主体


#### cookie

前面提到HTTP服务器是无状态的, 所以要想内容和用户身份联系起来就要使用cookie.

``` sequence-diagrams
client->server: request
server-->client: response + cookie
client->server: request + cookie
server-->client: response
```


#### Web缓存

Web缓存器也叫代理服务器. Web缓存器既是服务器又是客户端, 它会把请求的结果保存在本地. 在遇到相同的请求是可以由本地数据提供

可以减少网络负担. 如在高速的局域网络架设缓存器

HTTP协议有一种机制, 允许缓冲器证实它的对象是最新的. 这种机制叫做**条件GET**. 使用含有`If-Modified-Since: Date`请求行


### FTP

文件传输协议

区别于HTTP, FTP也运行在TCP上, 但是它使用两个并行的TCP连接来传输文件: 一个是**控制连接**, 一个是**数据连接**.


### 因特网中的电子邮件

由三个部分组成: 用户代理, 邮件服务器, 简单传输协议(SMTP).
邮件通过SMTP在邮件服务器中传递, 在由邮件服务器分发给对应用户.


### DNS

域名系统(Domain Name System, DNS)


### 套接字编程

#### TCP

#### UDP





