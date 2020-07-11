---
title: Python补完计划
date: 2020-2-28
tags: python
---


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


### 闭包

多层函数嵌套、往往内部函数用到外部函数的变量，一个特殊的对象

``` python
def func(a, b):
    def solve(x):
        print(a*x + b)
    return solve

ans = func(1, 2)
# 创造单独空间，包含参数a, b和solve函数。a, b相当于solve的全局变量
# 类似类，但比类开销小
ans(0)
ans(1)
ans(2)
```


#### 修改数据

``` python
def func():
    x = 100
    def solve():
        nonlocal x  # 告诉解释器x不是solve中的，否则由于x=10的存在。会导致解释器认为x这个局部变量在声明前使用
        print(x)
        x = 10  
        print(x)
    return solve
```


### 装饰器

有时候我们需要进行一下重复的过程, 比如计算函数用时. 如果我们直接把逻辑写在函数内部, 逻辑混乱且可读性不高。这时我们就可以使用装饰器


#### 装饰器的基本实现过程

``` python
def set_func(func):
    def call_func():
        func()
    return call_func

def test():
    pass

test = set_func(test)
test()

### 等价于

@set_func  # 等价于test=set_func(test)
def test():
    pass
```


#### 有参数的装饰器实现过程

``` python
def set_func(func):
    def call_func(a):  # 参数100会传到这
        func(a)
    return call_func

def test(num):
    pass

test = set_func(test)
test(100)
```


#### 不定参数的装饰器实现过程

``` python
def set_func(func):
    def call_func(*args, **kwargs):  # 参数会传到这，这里的星号是告诉解释器

        func(*args, **kwargs)  # 这里的星号是拆包!!!，否则就是一个列表、一个字典
    return call_func

def test(age, num, *args, &&kwargs):
    pass

test = set_func(test)
test(100)
```


#### 带有返回值的装饰器实现

``` python
def set_func(func):
    def call_func(*args, **kwargs): 

        return func(*args, **kwargs)  # 比包里调用，返回出去
    return call_func

def test(age, num, *args, &&kwargs):
    pass

test = set_func(test)
test(100)
```


#### 给装饰器的参数

``` python
def option(args):
    def set_func(func):
        def call_func(*args, **kwargs): 
            return func(*args, **kwargs)
        return call_func
    return set_func

@option(args)
def test(age, num, *args, &&kwargs):
    pass
```

装饰器需要一个函数指针，即@后跟函数名，由于option(args)不符合，所以先向下执行option(args)，返回的函数指针。@心满意足，用来装饰test函数


#### 多个装饰器对同一个函数进行装饰

先装下面的后装上面的。理解上面的实现过程。

执行效果是先执行上面的再执行下面的。所以装饰的顺序和想要的逻辑执行顺序相同即可。


#### 使用类当作装饰器

原理同闭包。只是变量名指向的不是函数，而是实例对象。

然后使用`变量名()`调用的是`实例对象.__call__()`

``` python
class Test(object):
    def __init__(self, func):
        self.func = func

    def __call__(self):
        return self.func()

@Test
def hello():
    pass
```

