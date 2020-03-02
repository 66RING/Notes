---
title: Python Tips
date: 2020-2-28
---

### 装饰器(Decorator)
有时候我们需要进行一下重复的过程, 比如计算函数用时. 如果我们直接吧逻辑写在函数内部, 逻辑混乱且可读性不高.
这时我们就可以使用装饰器

``` python
def a_new_decorator(a_func):
 
    def wrapTheFunction(*args):  # 包装的函数有很多参数时用*args代替
        # commands
        result = a_func()  # result接收了包装函数返回的结果
        # commands
        # return or not
 
    return wrapTheFunction
```

#### 使用
``` python
@Decorator
def func(*args):
    pass
```

