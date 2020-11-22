---
title: SQL注入
date: 2020-4-12
tags: SQL, database
---

## 原理

利用SQL拼接的安全性，将一些恶意的SQL语句传到服务器中执行。
如：

```
网页端登录页面：

    user: user
password: ****


传入服务器中可能就是
SELECT * FROM users WHERE id='user' and password='1234'
```

如果程序员没有进行过滤，则可能会发生这种情况

``` 
网页端登录页面：

    user: admin' or '1=1  # 单引号用来闭合SQL语句
password: ****


传入服务器中就是
SELECT * FROM users WHERE id='admin' or '1=1' and password='1234'
这样就注入了一个or true的语句，从而跳过password
```

SQL注入大同小异，但是每个数据库的SQL语句不一定相同，
当然现在SQL注入的门槛越来越高了。


## 注入分类

- 根据注入点类型分类
    - 数字型注入
        - `www.xxx.com/log?id=1`
        - `select * from table_name where id=1`
    - 字符型注入
        - `www.xxx.com/log?user=ring`
        - `select * from table_name where name='ring'`
        - 利用单引号闭合
    - 搜索型注入
- 根据提交方式分类
    - get注入
    - post注入
    - cookie注入
