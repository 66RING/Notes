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

#### TCP三次握手和四次挥手

- 三次握手：建立连接
    * C: 服务器你准备好没？SYN
    * S: 准备好了SYN ACK，你呢？SYN
    * C: 我也好了SYN ACK
- 四次挥手
    * C: 拜拜`close()`，不再给你发数据了哦
    * S: 好的(recv不等收了)知道了
    * S: 我也不发了(close，也许会延时，所以是4次)
    * C: 好的(recv不用等待接收了)
    * 服务器和客户端的接收都关了
- 为什么一般是客户端先关闭的原因：
    * 客户端的消息这么知道服务器收到了？服务器发生确认。那服务器的确认如何知道被收到了？客户端发送确认...无穷无尽也
    * 所以采用发起消息的一方设置超时时间(等待)，所以如果是服务器发起消息(close)那服务器就要进入等待(额外开销浪费)


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


### 协程方式

#### 迭代器

- 可迭代
    * `isinstance(ar, Iterable)`
    * 必须实现`__iter__`方法， **返回一个迭代器** ，`return Iterator(self)`，如果这个类本身是个迭代器(实现了`__next__`)则可以返回self
- 迭代器
    * `isinstance(ar, Iterator)`
    * 必须实现`__iter__`方法和 **`__next__`方法**
    * 使用`raise StopIteration`来表示迭代结束
    * `iter(Iterable)`方法气质其实调用了类的`__iter__()`，返回类的迭代器，for in就是调用了`__iter__()`获得迭代器的
    * `next(Iterator)`方法调用迭代器的`__next__()`方法
    * 转换成list转化成tuple也是使用了迭代器

迭代器储存生成这个数据的方式，而不是生成这个数据的结果。占用极小的内存空间


#### 生成器

生成器是一种特殊的迭代器

- 列表式创建生成器：`nums = (x for x in range(10))`
    * 不同于`nums = [x for x in range(10)]`直接返回结果，生成器返回生成数据的方式
- 函数变成生成器
    * 使用`yield`返回生成器对象
        + yield相当于返回值后把函数暂停，下次使用将从上一次的位置继续向下走
        + 获取生成器return的结果，一般捕获异常，`except StopIteration as ret`，然后使用`ret.value`取得返回值
- 通过`send`启动生成器：`gen.send(args)`
    * 同一会执行一次`__next__`，但区别于`next`，`send`能往里传参数
    * `ret = yield x`，由于`yield x`返回给外面，没有返回值，所以`ret=None`。send传递的参数args使得`ret=args`
        + 一般用于更新状态
    * 第一次迭代使用send会出错


#### 使用yield实现多任务

使用yield把普通函数转化成生成器，这样对于一个含有无限循环的函数，每轮yield后就会暂停，让下一行代码执行。这就实现了用函数实现并行。

但不同于操作系统级的并行(上下文切换开销相当大)，这样的多任务就像使用一个函数一样简单。

开销：$进程>线程>协程$

但这样存在一个问题，如果有100个这样的函数要写100个调用？这时需要使用`gevent`

geven是一个基于协程的并发库

- `greenlet`：对yield进行了封装
    * 创建greenlet对象：`gr1 = greenlet(func_pointer)`，对函数进行封装，传入普通函数就行，不需要yield返回了。
    * 切换：`gr1.switch()`切换到gr1(封装的函数)
        + 如果在函数1中`switch`到函数2，函数2switch到函数1，就和yield效果一样
- `gevent`：对greenlet进行封装
    * 创建：`g1 = gevent.spawn(func, arg1, arg2, ...)`
    * 特点：遇到`gevent.sleep()`会切换，greenlet遇到延时会等待
        + 因此在结尾写上`g1.join()`(等待g1执行完成，实现了geven.sleep)，使它遇到了延时，它就自动切换执行
        + `geven.joinall([list])`，把所有geven对象放入列表就会等待列表内所有
    * `monkey.patch_all()`，是否所有的延时都要手动换成`geven.sleep`？
        + 使用`monkey.patch_all()`它会自动将延时换成`geven.sleep`，包括网络延时


## HTTP

报文格式
- 请求行
    * 方法 URL 版本
    * 包含一系列必要信息
- 首部行(optional)
    * 首部字段: 值
    * 相当于指定配置
- 空行
    * 通过空行来分割头和body
    * 浏览器通过`\r\n`解析换行
- 实体主体

``` 
--- 请求行 ---
GET /some/dir HTTP/1.1
--- 首部行 ---
Host: www.some.com           # 指明对象所在的主机, 该首部行的Web高速缓存所要求的
Connection: close            # 非持续连接
User-agent: Mozilla/5.0      # 发送请求的浏览器类型
# 还会有很多的配置，首部行是连续的，如果遇到空行则说明首部行结束，进入body
```

当请求一个页面的时候，返回html，html不会有图片(css、视频或且它等)的数据但会有图片的超链接。浏览器解析到超链接会自动发送一个新的请求，因此可以从一个请求中生出很多个请求。

- 多进程web服务器
    * 因为子进程会复制父进程的资源，所有子进程里close了父进程里还有close
- 多线程web服务器
    * 如果是使用多线程Thread，不会复制父进程资源，子线程里close了父进程就不需要close了

p74





