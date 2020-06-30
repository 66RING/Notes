---
title: python 网络编程
date: 2020-6-25
tags: python, network
---

## Socket基本使用

- 创建套接字对象`socket.socket(AddressFamily, Type)`
    * udp: `s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)`
    * tcp: `s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)`
    * 其中`AF_INET`表示IPv4，`SOCK_DGRAM`表示udp，`SOCK_STREAM`表示tcp
- 绑定端口`s.bind((ip, port))`让程序使用固定的端口负责接收
- 发送数据
    * udp: `s.sendto(data, (ip, port))`，如果没绑定端口则由系统随机分配
    * tcp: `s.send(data)`
- 接收数据
    * 1. 创建套接字
    * 2. 必须绑定端口，让程序使用固定的端口负责接收
    * 3. 接收`s.recvfrom(size)`，返回元祖`(data, src_ip)`

- 单工：只能收/发
- 办双工：能收发，要收就不能发，要发就不能收
- 全双工：能收发，如打电话

套接字可以同时收发，是全双工


### Socket中使用tcp

和udp的方法有些不同

- 建立连接
    * `s.connect((ip, port))`
- 发送数据
    * tcp: `s.send(data)`


### tcp服务器

一般需要以下步骤：
- 创建套接字
- bind绑定ip和port
- listen使套接字变为被动链接
    * 被动：即可以让别人来呼叫
- accept等待客户端的链接
    * 返回一个元祖，`(client_socket, client_addr)`
        + 可以通过`client_socket`向客户端发送数据
- recv/send收发数据
    * 不同与recvfrom，recv只返回data，因为accept的时候已经知道来源地址了

- 同时为多个客户端服务，并且多次服务一个客户端
    * 由于accept会阻塞进程(类似input)，如何同时为多个服务？
        + 多任务
    * 一个客户端不一定只需要一次服务，如何多次服务一个客户端？
        + 当一个客户端不需要服务的时候，即客户端调用close的时候，会使服务端recv解堵塞，返回空，以此做判断依据。


### 一个简单下载器的实现

- 服务端
    * 根据用户需求寻找文件
    * 将文件数据发送给客户
- 客户端
    * 在本地新建一个同名文件
    * 将从服务接收(recv)到的数据写入(二进制模式)文件即完成下载任务


### TCP注意点

- tcp服务器一般绑定端口
    * 否则客户端找不到服务器
- tcp客户端一般不绑定端口
    * 因为连接服务器只需服务器的ip、port等。而且如果绑定不利于客户端多开
- tcp服务通过listen将socket创建出来的主动套接字变为被动是必要的
- 关闭listen后的套接字意味着被动套接字关闭了，新的客户端不能连接，但之前已经连接成功的客户端正常通信


## 多任务

### Thread的基本使用

对于单核cpu，可以在程序间快速切换，以达到多任务的效果，如：qq执行0.00001s，然后切换到微信。如此快速切换以致于人无法察觉，来实现多任务。但本质操作系统调度，一次执行一个任务，这样的方式称为并发。

对于多核cpu，每个核功能同单核，多个核在可以同时多个任务。这些在不同核中执行的进程的方式称为并行。真正意义的多任务。

thread使用需要引入threading模块：`import threading`

``` python
import threading

t1 = threading.Thread(target=func_pointer1, args=(args,))  # 返回对象，参数以元祖方式传入
t2 = threading.Thread(target=func_pointer2, args=(args,))
t1.start()
t2.start()
```

创建子线程t1、t2，通过start()来开始执行子线程，开启后执行下一行，子线程在后台执行，子线程执行完后主线程关闭。这样就实现了t1、t2多任务。

- `threading.enumerate()`
    * 以列表形式返回正在运行的线程
    * 查看线程数量:`len(threading.enumerate())`
- `t.start()`
    * start实际上是调用了`threading.Thread`的`run(self)`方法，所以可以通过继承然后重载run方法
- 多线程间是共享全局变量的
    * 同时对一个数据进行写入操作会存在问题，因为写入操作大致可分为这么几个步骤：读取数据、操作数据、写入数据。由于cpu在线程间切换，如果操作数据后切换到另一个进程，然后才写入数据，将导致问题。
    * 互斥锁解决资源竞争
        + 创建互斥锁：`mutex = threading.Lock()`
        + 锁定：`mutex.acquire()`，如果已经上锁会阻塞直到解锁
        + 释放：`mutex.release()`
    * 多个互斥锁会出现的问题
        + 死锁：线程A等线程B用完资源，偏偏资源B在等A用完资源，的状态叫做死锁，如
        ``` python
        def funcA():
            mutexA.acquire()
            mutexB.acquire()
            mutexB.release()
            mutexA.release()
            
        def funcB():
            mutexB.acquire()
            mutexA.acquire()
            mutexA.release()
            mutexB.release()
        ```
        + 避免死锁：
            + 程序设计时尽量避免(银行家算法等)
            + 添加超时时间


### 进程方式

需要`multiprocessing`模块

``` python
import multiprocessing
p = multiprocessing.Process(target=func_pointer, args=(args,))
p.start()
```

- 由于是进程，所有可以在shell中查看`ps -aux`，也可以进行任何进程操作
    * 但是耗费的资源也比线程大
- 进程间是互相独立的，进程间如何通信呢？
    * 队列，直接在内存中操作
        + **multiprocessing.Queue()**
        ``` python
        q = multiprocessing.Queue()
        q.put(arg)
        q.get()
        q.get_nowait()
        q.full()
        q.empty()
        ```
        + 当队列为空时get方法会阻塞，get_nowait方法会引发异常来告诉你队列为空
        + 如果队列满了put方法会阻塞
    * 使用队列来减少进程的耦合(解耦)
        + 进程A往queue填数据，只需关注写
        + 进程B从queue取数据，只需关注读


#### 进程池

一个可以容纳很多进程的特殊容器。它 **重复利用进程池里的进程** ，因为进程并非越多越好，会增加操作系统调度的压力。进程池减轻了进程创建和销毁的负担。

- 进程池创建
    * `from multiprocessing import Pool`
    * `po = Pool(n)`，设置最对创建的个数，这里是n
        + 可以往里添加无数个，但最对同时运行n个进程，其他存起来
    * 添加到进程池：`po.apply_async(func_pointer, (args,))`
    * 进程池关闭`po.close()`
    * `po.join()`等到进程池中所有子进程执行完成， **join必须放在close之后**
        + 因为使用进程池不会阻塞主进程，所以可能子进程还没结束，主进程先结束了，导致所以子进程结束
- 进程池里的任务产生的异常不会产生错误信息
- 显示进度
    * 通过队列在进程间通信，子进程完成后写入队列，主进程从队列读，统计
        +  **进程的队列要使用 `multiprocessing.Manager().Queue()`**


P46

### 协程方式




