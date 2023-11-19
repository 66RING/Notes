---
title: 聊天应用的系统设计
author: 66RING
date: 2023-11-04
tags: 
- system design
- interview
mathjax: true
---

# 聊天应用的系统设计

- ref
    * https://www.youtube.com/watch?v=xyLO8ZAk2KE&ab_channel=ByteMonk
    * https://medium.com/@m.romaniiuk/system-design-chat-application-1d6fbf21b372
    * https://bytebytego.com/courses/system-design-interview/design-a-chat-system
- abs
    * nosql
    * 对象存储
    * multi service
    * 消息队列(MQ)


## 功能需求分析

- one-one chat
- group chat
- read receipt(已读)
- online status
- share multimedia(分享)

## 系统需求分析

- 低延迟
- 高可靠
- 高可用
- 多平台
- 历史记录
- 处理大文件的能力(BLOB store)
- 端到端加密
- web socket(处理不会立即断开的连接)


## 性能评估

- 总活跃用户: 500M
- 每人每天30个消息
- 每天总消息数: 500M * 30 = 1.5B
- **每秒**消息数: 1.5B / (3600 * 24)


## 存储评估

- 每天总消息数: 1.5B
- 每条消息大小: 50KB
- 每天存储消耗: 1.5B * 50KB = 75pb

## API设计

- `send_msg(sender_id, reciever_id, text)`
- `get_msg(user_id, screen_size, before_timestamp)`

### services

- messaging
    1. 场景: A send to B
    2. A向messaging service发送信息, 通过websocket建立持久链接
    3. messaging service通过session service找到B, 并将消息发送给B 
    4. 如果通过session service发现用户不在线, 则可通过relay service暂存数据
- session
    * 用户访问了msg service, msg service告诉session service哪个用户建立的连接
    * session service之后保存数据到数据库(通常是nosql)
- relay
    * 当对方不在线, 需要先暂存消息(from, to, msg), 存储到数据库(如Cassandra)
- last seen
    * client端发送以保存用户状态信息
- asset
    * 用于保存和访问多媒体数据, 保存到**对象存储/blob**, e.g. AWS s3
- group
    * 广播到组内所有用户(消息队列MQ), 根据session service识别组内有哪些用户

### db schema

- User
- Group
- Unsend msg
- Last seen
- Session
    * (user id, service id)


### 群聊执行流程

> A发消息到Group1

1. A的websocket向msg service发送消息
2. msg service保存消息到消息队列
    - msg service作为生产者, group msg handler作为消费者
    - group msg handler查询组内的所有用户
    - group msg handler通过websocket manager找到用户的通信方式, 然后向对应websocket handler发送消息
    - websocket handler
        * 负责处理发送, 包括用户不在时保存到relay service


### 媒体文件存储

> asset service -> 对象存储/AWS S3

1. UserA update image, get url/id
2. UserA send url/id to UserB
3. UserB fetch image by url/id

