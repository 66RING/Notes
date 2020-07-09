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
- 不论是多进程还是多线程，都要创建一个子进程/线程，如果有很多人同时请求服务(如双11)开销相当大
- 使用协程的web服务器
    * 1. 使用gevent
    * 2. 非阻塞方式使用socket实现单进程单线程监听多个套接字
    ``` python
    s.setblocking(False)  # 使用非阻塞方式
    client_socket_list = list()
    while True:
        try:
           new_socket, new_addr = s.accept() 
        except Exception as ret:
            print(ret)
        else:
            new_socket.setblocking(False)  # 新socket使用非阻塞方式
            client_socket_list.append(new_socket)
     
        for client_socket in client_socket_list:
            try:
                recv_data = client_socket.recv(1024)
            except Exception as ret:
                print(ret)
            else:
                if recv_data:
                else:
                    # 对方关闭了
                    client_socket.close()
                    client_socket_list.remove(client_socket)
    ```
        + 但这么做列表越大效率越低


### 长连接

- 长连接
    * 用同一个连接获取数据
    * http1.1
- 短连接
    * 获取一个数据建立一个连接
    * http1.0

我们之前每一轮都关闭连接，虽然说是HTTP1.1，实际我们一直用短连接的方式传输

短连接模式，浏览器通过接受close知道包的内容范围。但如果使用长连接，连接不close，浏览器如何知道包的范围？

使用长连接，需要在header中标注`Content-Length`，告诉浏览器包的长度，这样浏览器在获取全部内容后主动断开连接，服务器再断开。


### epoll

在单进程单线程服务多用户的实例中，使用列表把所有连接存储起来。但这样随着列表变大，效率将变低。因为需要拷贝fc(文件描述符)到内核的内存空间，这样轮循+拷贝的方式效率相当低。

epoll有个特殊的内存空间，操作系统和应用程序共用。在这个内存中的所有要监听的套接字检测时不采用轮循而是方式事件通知的方式。

``` python
import select  # 引入模块

# 创建epoll对象
epl = select.epoll()  # 创建这样的共享内存

# 将监听套接字对应的fd注册到epoll
epl.register(your_socket.fileno(), # fileno()返回fd
            select.EPOLLIN)  # EPOLLIN表示监听是否有输入

epl.poll()  # 默认阻塞，知道OS检测到数据到来，通过事件通知的方式告诉程序，才会解阻塞
# 返回列表，一次通知多个[(fd, event), (套接字对应的文件描述符, 对应的事件)]
```

## python提高

### GIL(全局解释器锁)

保证多线程程序同一时间只有一个线程在执行。多个线程先强锁。

c语言写的python解释器存在GIL。

一面试题
> 描述python GIL的概念，以及它对python多线程的影响。编写一个多线程抓取网页的程序，并阐明多线程抓取程序是否比单线程性能有提升，并解释原因

参考答案
> - 1. python语言和GIL没有半毛钱关系。仅仅是由于历史原因在Cpython解释器，难以移除GIL
> - 2. GIL：全局解释器锁。每个线程在执行的过程都需要先抢GIL，保证同一时刻只有一个线程可以执行
> - 3. 线程释放GIL锁的情况：在IO操作等可能会引起阻塞的system call之前，可以暂时释放GIL，但在执行完毕后，必须重新获取GIL python3.x使用计时器(执行时间到达阀值后，当前线程释放GIL)或python2.x的tickels计数到100
> - 4. python使用多进程可以利用多核CPU资源
> - 5. 多线程爬取性能有提升，因为遇到IO阻塞(如网络)会自动释放GIL锁

IO密集型程序适合用多线程


### 深拷贝、浅拷贝

赋值语句在python中一般都是引用

- 深拷贝`copy.deepcopy`
    * `import copy`
    * `b = copy.deepcopy(a)`
    * `id(a) != id(b)`
    * 如果拷贝的是元祖，且元祖里有可变的数据，设元祖a，则deepcopy结果`id(a)!=id(b)`
- 浅拷贝`copy.copy`
    * `import copy`
    * `b = copy.copy(a)`
    * `id(a) != id(b)`
    * 但是如果拷贝的是元祖，且元祖里只有普通数据(不可变的)，设元祖a，则copy结果`id(a)==id(b)`
        + 因为元祖是不可变类型，增删改都没用所以拷贝有什么用，所有就不拷贝
- 切片也是浅拷贝
- 字典`key: value`，value是指向别处的引用
- 浅拷贝和深拷贝的区别
    ``` python
    a = [1, 2]
    b = [3, 4]
    c = [a, b]  # c中的a、b都是引用，引用指向两个列表
    d = copy.deepcopy(c)
    e = copy.copy(c)
    # 虽然id(c)!=id(e)但是e中的[1, 2]、[3, 4]仍是a、b的引用，仅仅是把c的东西原封不动复制到e
    ```


### 私有化

不同于面向对象的语言，python没有public、private等关键字。

- xx：共有变量
- \_x：单前置下划线，私有化属性或方法，`from somemodule import *`不会导入`_x`变量，类和对象子类可以访问
- \_\_xx：双前置下划线，私有化属性或方法，避免与子类中的属性冲突，无法在外部直接访问(名字重整所以访问不到)
- \_\_xx\_\_：双前后下划线，用户名字空间的魔法对象属性，非私有
- xx\_：单后置下划线，用于避免与python关键词的冲突


### import问题

程序执行时添加新的模块路径

`sys.path`是个储存了模块路径的列表，因此可以使用列表操作改变搜索路径的优先级以及添加新路径


#### 重新导入模块问题

import会防止模块重复导入，如果在程序执行期间修改了模块，即使使用import再次导入，修改的模块不会更新。需要使用

``` python
from imp import reload

reload(somemodule)  # 使用这种方式在不退出程序的情况下重新导入模块

# 但是对于from aa import bb这样的需求没有办法
```


#### 多模块导入问题

在大型项目中一般会把很长的代码拆分成很多小的模块，这时模块间的数据传递就需要注意。一般把公共数据放在一个模块，这样方便访问、修改。

`import aa`，`aa.bb = a`和`from aa import bb`，`bb=a`的区别

- `import aa`使aa指向模块，则`aa.bb = a`是对模块aa的bb赋值，会改变aa中bb的值
- `from aa import bb`使得变量bb **指向** 模块aa中的同名变量bb，如果使用`bb = a`使得bb的指向改变，不会改变aa中的bb的值


### 面向对象

#### 多继承以及MRO顺序

- 调用父类方法的方式
    * 1. 通过父类的名字调用
        + 缺点是会根据类递归的调用，无形中造成资源浪费。如
        ``` python
        Class A:
            __init__(self):
                new_socket
        Class B(A):
            __init__(self):
                A.__init__(self)
        Class C(A):
            __init__(self):
                A.__init__(self)
        Class D(B, C):
            __init__(self):
                B.__init__(self)
                C.__init__(self) 
        # B和C的init分别调用A的init导致多创建一个socket，造成浪费
        ```
    * 2. 通过`super().xxx`调用
        + 不是更具类递归的调用，而是根据`ClassName.__mro__`中的顺序调用，保证了每个类只调用一次
        + 如果多继承了多个同名方法，则根据`ClassName.__mro__`中的顺序决定super().xxx调用的是哪个(先后顺序)
        ``` python
        Class A:
            __init__(self):
                new_socket
        Class B(A):
            __init__(self):
                supter.__init__(self)
        Class C(A):
            __init__(self):
                super.__init__(self)
        Class D(B, C):
            __init__(self):
                super().__init__(self)
        # 其中print(D.__mro__)=(D, B, C, A, object)
        # 那么如果从D开始，如果父类都有调用super，则会根据mro中的顺序调用，即D、B、C、A
        ```
        + `super(ClassName, self)`，会从ClassName往后开始调用，如`super(B, self)`则顺序是B、C、A。默认从当前类开始


#### 可变参数

- `func(a, *args, **kwargs)`
    * 一个`*`号以元祖的形式传递参数，变量名是args，`*`号只是告诉编译器
    * 两个`*`号以字典的形式传递参数，变量名是kwargs
        + **接收关键字参数** ：如`func(1, 2, 3, 4, age='12', name='ring')`
            + args=(2, 3, 4)
            + kwargs={'age': '12', 'name': 'ring'}
        + 需要注意的是如果传的是一个字典，它并不是关键字参数，而是一个字典(一个整体)


#### 静态方法和属性方法

- 类对象和实例对象
    * 创建一个对象会从模板类中调用`__new__`分配内存空间，`__init__`初始化内存空间，`__class__`指向创建这个实例对象的类对象
    * 对于公有的方法、属性存储在类对象中
        + 如方法`__inti__(self)`就不必每个实例都有一份，放在类对象中即可
    * 对于特有的方法、属性存储在类对象中
        + 如初始化name=ring，那么对于这个实例的name是ring，别的实例有所区别
- 类方法、实例方法、静态方法
    * 实例方法：一般的方法
        + 很难修改类属性，若`obj.class_state="xx"`原来`class_state`是一个类属性。这个方法将导致实例里面新增一个名为`class_state`的属性
            + 要修改也是可以的`obj.__class__.class_state="xx"`就可以修改
        ``` python
        class A:
            # 实例方法
            def func(self):  # 默认传实例对象的引用self
                pass
        ```  
    * 类方式：用`@classmethod`装饰
        ``` python
        class A:
            @classmethod  # 类方法
            def func(cls):  # python解释器默认把类对象引用cls传入
                pass
        ```
        + 可以修改类属性
    * 静态方法：用`@staticmethod`装饰
        + 相当于在类外定义一个函数， **不让python解释权默认传入类对象或实例对象** 。写在类中是为例在不同类中区分开来


#### property属性

- 用装饰器创建
    * 让代码更简洁，调用一个函数像取值、赋值一样
    * 在普通方法前用`@property`修饰，如。把调用方法改成"调用属性"，但实际还是调用方法，只是可读性更高
        ``` python
        class A:
     
            @property
            def func(self):
                return 0  # 必须返回一个值，且参数只有self

        a = A()
        a.func  # 可以通过a.func调用，而不用a.func()
        ```
    * 新式类(继承object，python3默认继承)中有3中property装饰器
        ``` python
        class A:
     
            @property
            def func(self):
                return 0   # 获取值
     
            @property.setter
            def func(self, value):  # 要同名，且传入新值value
                print("some")  # 设置值
            # 如可以调用xxx.func = 100
  
            @property.deleter
            def func(self):
                print("some")  # 删除值
            # 如可以调用del xxx.func
        ```
- 通过类属性创建
    * `property(arg1, arg2, arg3, arg4)`
        + 参数1是方法名，调用`对象.属性`时自动触发执行
        + (可选)参数2是方法名，调用`对象.属性=xx`时自动触发执行
        + (可选)参数3是方法名，调用`del 对象.属性`时自动触发执行
        + (可选)参数4是字符串，调用`对象.属性.__doc__`时此参数是该属性的描述信息
    ``` python
    class A:
 
        def func(self):
            return 0 
        
        FUNC = property(func)

    a = A()
    a.FUNC 
    ```

#### 修改私有属性

私有属性(在以`__`开头的变量)之所以无法访问是因为python悄悄改了变量名。如把`__func`改成了`_className__func`。所以使用这个改后的名就可以访问私有属性。这机制叫做名字重整。


#### 魔法属性/方法

- `__doc__`和`help()`
    * 使用`var.__doc__`或`help(var)`可以查看写在开头的描述
- `__module__`和`__class__`
    * `__class__`表示当前操作的对象的类是什么
    * `__module__`表示当前操作的对象是在哪个模块
- `__init__`
    * **初始化** 方法，创建类对象时自动触发执行
- `__del__`
    * 对象释放时，自动触发执行
- `__call__`
    * 对象后面加括号，触发执行
    ``` python
    obj = classA()
    obj()   # obj.__call__()
    ```
- `__dict__`
    * 类或对象的所有属性
- `__str__`
    * 如果一个类中定义了`__str__`方法，那么打印对象时，默认输出改方法的返回值
- `__getitem__`、`__setitem__`、`__delitem`
    * 如果类中实现了这3个方法，则可以当字典用
    ``` python
    class A:
        def __getitem__(self, key):
            print(key)
         
        def __setitem__(self, key):
            print(key)
         
        def __delitem__(self, key):
            print(key)
    
    odj = A()
    res = obj['k1']   # __getitem__
    obj['k2'] = 'abc' # __setitem__
    del obj['k3']     # __delitem__
    ```
- `__getslice__`、`__setslice__`、`__delslice__`
    * 如果类中实现了这3个方法，则可以用于分片操作，如列表
    ``` python
    class A:
        def __setslice__(self, i, j):
            pass
         
        def __setitem__(self, i, j):
            pass
         
        def __delslice__(self, i, j):
            pass
    
    odj = A()
    obj[-1:1]            # __getslice__
    obj[0:1] = [1, 2, 3] # __setslice__
    del obj[0:2]         # __delslice__
    ```


#### with与上下文管理器

使用with打开文件能够保证最终文件都会关闭。如果采用传统的`f = open()`则需要try-catch辅助。with是一种更简洁的写法。


- 上下文管理器
    * 任何实现了`__enter__()`和`__exit__()`方法的对象都可称之为上下文管理器。
    * `__enter__()`返回资源对象
    * `__exit__()`处理一些清理工作

当一个对象实现了上下文管理器，就可以使用with语句了

``` python
with obj(args) as f:  
    # obj()创建实例对象
    # with自动调用了obj(上下文管理器)的__enter__方法，enter的返回值赋给f
    # 如果产生了异常，将自动调用__exit__方法
```


p116













