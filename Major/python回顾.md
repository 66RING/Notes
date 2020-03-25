# python回顾

### 2 vs 3:

​		1.//p2p3中表示整除；/3是真除，p2是整除(浮点型自动真除);;from __future__ import division 可在py2中用py3的除法

​		2.p2的input会自动识别输入的类型，也可用raw_input()来让它一律字符串类型；而p3中就一律都是字符串

​		3.23.print

```python
#p2 print:
	print i
	print i,#不换行
#p3 print:
	print(i)
    print(i,end='')#不换行

```

​		4.py2支持不等于 <>，而3不支持

​		5.py2=>str,unicode;py3=>bytes,str

​		6.py2: next()  py3: __next__()

### map:

```python
map(function, iterable, ...)
```

​		map() 会根据提供的函数对指定序列做映射。
​		第一个参数 function __以参数序列中的每一个元素调用 function 函数，返回包含每次 function 函数返回值的新列表__。		py2中直接返回列表；py3中返回的是一个可迭代对象

### append() vs extend():

​		append直接把object加在后，extend把可迭代对象中的元素挑出再加入

### 切片操作[:] [::]

### zip

```python
a = [1,2,3]
b = [a,b,c]
d = zip(a,b)
>>>d
[(1,a),(2,b),(3,c)] #py2中，直接返回，py3中返回可迭代对象
```

​		将多个列表对应位置的元素组合成元祖，返回于map类似	

### 列表推倒式

```python
a = [x for x in range(10) if x%2==0]
#相当于
for x i range(10):
    if x%2 == 0:
        a.append(x)
```

### 字典

​		用已有数据创字典的方法：
​				1.a.dict(key1=value1,key2=value2,......)  #不能同int作为键
​				2.a.dict(zip(keys,values))

​		字典读取键对应的值：
​				1.用下标 d["key"]
​				2.用get   d.get("key")

​				py2中，items(),keys(),values()返回列表；py3中返回可迭代对象

​		字典元素的添加：
​				1.d.update(spam=111) #加一个
​				2.d.update({'aa':12,'bbb':2}) #加一个字典
​				3.d.update(('aa',1),)   d.update(['aa',12],['bb',12]) #同上

### 函数

​		（什么情况下形参影响实参）

​		默认值参数
​				1.默认值放末尾
​				2.默认值为空列表时的危险

​		关键参数
​				fun(a=1,b=2) #对应给出就不用在乎顺序了

​		可变长参数
​				1.*p  接受多个实参并将其存放在一个元组里
​				2.**p 接受字典形式的实参

​				__ps:*和**出现在函数调用中表示对参数解包 如fun(*[a,b,c,d])就是解开然后加进去，**同理，字典的一一对应__

​		__lambda表达式 (参数):(操作)__

### reduce

​		将一个接收两个参数的函数，从可迭代对象中从左到右取数据进行操作
​		py3中要from functools import reduce	

```python
reduce(function, iterable[, initializer])
>>>def add(x, y) :            # 两数相加
...     return x + y
... 
>>> reduce(add, [1,2,3,4,5])   # 计算列表和：1+2+3+4+5,乘法也是同理
15
>>> reduce(lambda x, y: x+y, [1,2,3,4,5])  # 使用 lambda 匿名函数
15
```
### 迭代

​		1.iter()从可迭代对象中获取迭代器
​		2.生成器函数(一种特殊的迭代器)

~~~python
def func():
	yield 1
	yield 2
	...
~~~

​		3.生成器表达式(make an iterable)
​				(x**2 for x in range(4)) #注意是用圆括号 ps没有元组推倒式

### 面向对象

​		1.类的定义(新式类旧式类)
​				class A(父类):
​		2.常用方法.....
​		3.实例方法
​				1.self参数，用来表示创建的实例对象本身
​				2.重载
​		4.静态方法
​				没有self参数，可用类名或实例名访问				
​				1.装饰器定义 @staticmethod
​				2.staticmethod()定义		
​		5.类方法
​				cls为第一个参数，表示类对象；功能类似self
​				1.装饰器定义  @classmethod

​		6.super()
​				1.super(x,self).func():从x类向上找，找到后把实例self多态变，然后用那个func()

​		7.抽象超类...

### 异常处理

~~~python
try:
	pass
except Exception:
	pass
else:
~~~

​		raise手动产生异常

​		with...as... 最终必会执行清理行为

### 常用模块

​		pickle;random;chardet;sys;time;math

​		


