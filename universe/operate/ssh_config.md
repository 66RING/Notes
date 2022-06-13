---
title: ssh配置免密和别名
author: 66RING
date: 2020-06-05
tags: 
- tags
mathjax: true
---

# ssh配置免密和别名

刚接触ssh时一般都是命令行输入完整用户和服务器ip，然后再输入登录密码完成等。这样非常麻烦，所以就有了这篇文章，配置免密登录和为服务器配置别名然后直接`ssh <name>`登录

1. `ssh-keygen`生成密钥对
2. 使用ssh-copy-id工具将公钥发送到服务，这令格式如下
	- ssh-copy-id -i ~/.ssh/id_rsa <user>@<ip>
	- 其中`~/.ssh/id_rsa`就是`ssh-keygen`生成的公钥文件

经过如上操作，应该就可以免密登录进服务器了。那么接下来就是压缩命令要敲的命令了。要实现`ssh ali`就能登录到服务器的效果

要为服务器设置别名使得可以`ssh ali`直接登录需要修改`~/.ssh/config`文件，然后按照如下格式填写内容

```
Host  ali
	User  <user>
	Hostname <ip>
	port 22
	IdentityFile ~/.ssh/id_rsa
```

每个别名配置都是这样一个代码块，其中

1. `Host ali`中`ali`就所起的别名，之后可以通过`ssh ali`
2. `User <user>`中`<user>`就是你要登录的用户的名称
3. `Hostname <ip>`中`<ip>`就是服务器的ip地址
4. `IdentityFile ~/.ssh/id_rsa`中`~/.ssh/id_rsa`就是登录所需的密钥文件

之后在命令直接`ssh ali`就等价于`ssh -p 22 <user>@<ip>`并输入密码了
