---
title: lua学习笔记
date: 2020-7-23
tags: lua
---

## 概述

类似python等脚本语言，在循环、方法、条件等语句需要使用end表示结束，因此不像python那样依靠缩进。

- 解释器执行`lua <filename>`
- 编译后执行：(不透露源码)
    * `luac x.lua`
    * `lua x.out`
- lua使用`..`连接字符串


## 基本使用

### 变量

``` lua
a = 100         -- 全局变量
local b = 100   -- 局部变量，不影响全局
print(a)
```

### 方法

``` lua
function new_function(args)
    
end
-- 同python动态类型，同bash需要end所以不依靠缩进
```


### 运算符

- 加减乘除正负没差
- `^`求幂，如`2^100`2的100次幂


#### 关系运算符

- 大于等于小于没差
- `~=`，**不等于**
- 与或非，`and`, `or`, `not`
- `..`，连接两个字符串
- `#`，一元运算符，返回字符串长度


### 条件
``` lua
if a>b then
    return a
elseif a=b then
    return a
else
    return b
end
```


### 循环

``` lua
for var=1, 100 do
    print(var)    -- 打印1到100
end
```


### 表(字典) 

``` lua
Config = {key1=value1, key2=value2}   -- 创建一个表
Config.word = "Hello"   -- 往表里增加元素
Config.num = 100
Config["name"] = "ring"   -- 往表里增加元素
print(Config.key)      -- 使用
print(Config["key"])
```

- 遍历
    ``` lua
    for key, var in pairs(Config) do
    end
    ```


### 数组

lua本身并没有数组的概念，使用的是上面列表然后配合lua的一些api

- 创建`arr = {1, 2, 3, 4, "hello"}`
- 遍历，lua数组的索引从1开始
    ``` lua
    for key, var in pairs(arr) do
        print(key, var)  -- 数组的索引从1开始
    end
    ```
- 通过api操作
    ``` lua
    arr = {}
    for var=1, 100 do
        table.insert(arr, 1, var)
        -- table.insert(table, pos, value)
        -- 在table的pos位置插入value
        -- 所以上面的插入索引和value顺序相反
        -- 除非table.insert(arr, var, var)，则索引==value
    end
    ```
- 数组长度`table.maxn(arr)`，**注意lua数组索引从1开始**


### 面向对象

lua本身没有面向对象，但是可以通过某种方式实现

- 使用表的形式(类似golang)
    ``` lua
    Obj = {}
    function Obj.sayHi()
        print("hi")
    end
    -- 或者
    Obj.sayHi = function()
        print("hi")
    end
    ```
    * 创建实例，由于lua的类实质就是一个表，所以遍历表，复制，返回即可
        ``` lua
        function clone(tab)
            local ins = {}
            for key, var in pairs(tab) do
                ins[key] = var
            end
            return ins
        end
        -- 然后这里创造了一个类Obj，balabala
        local p = clone(Obj)
    ```
    * 构造方法，书接上文
        ``` lua
        Obj.new = function(arg)
            local self = clone(Obj)
            self.name = arg

            return self
        end
        local p = Obj.new("ring")
        ```
    * 关于指向自己的指针
        ``` lua
        Obj.hello = function(self)
            print("hello "..self.name)  -- ..表示连接字符串
        end
        local p = Obj.new("ring")
        p.hello(p)
        p:hello()   -- 默认传入自己作为第一个参数
        -- 两个方法等效，一般使用下面这个方法
        ```
    * 继承
        ``` lua
        function copy(dist, tab)
            for key, var in paires(tab) do
                dist[key] = var
            end
        end
        Son = {}
        Son.new = function(arg)
            local self = Obj.new(arg)
            copy(self, Son)  -- 也别忘了自己原有的东西
            return self
        end
        ```
- 使用函数闭包的形式面向对象，可以实现私有化，但是表的方式更块些
    ``` lua
    fuction Obj(arg)
        local self = {}
        local function init()   -- 私有
            self.name = arg
        end

        self.sayHi = function()
            print("hi")
        end

        init()  -- 标准化
        return self
    end
    local p = Obj("ring")
    p:sayHi()
    ```
    * 继承
        ``` lua
        function Son(arg)
            local self = Obj(arg)
            -- some
            return self
        end
        ```
- 使用**元表**
    * 语法糖`obj:funciotn()`，默认会传入self
    * 模板
    ```lua
    function A:New()
        local o = {}
        setmetatable(o, self)  // self作为o的元表，仅仅建立起关联，这时self中的任何值都不会影响到o
        self.__index = self    // 当元表中存在__index时，如果key索引不到，会从__index找，因此这步开始才让o和self有影响。这样就实现了o继承self，即A
        return o
    end
    ```


# Lua高级编程

## metatable元表

我们无法直接对table进行操作，lua提供元表允许我们改变table的行为，每个行为关联对应的元方法

当lua试图对两个table进行操作时，先检查两者之一是否有元表，然后调用对应的方法

- `setmetatable(table, metatable)`，把metatable设置位table的元表
- `getmetatable(table)`，获取table的元表

```lua
mytable = {}
mymetatable = {}
setmetatable(mytable,mymetatable) -- 把 mymetatable 设为 mytable 的元表

-- 也可一行实现
mytable = setmetatable({},{__index=xxx, _newindex=xxx, ...})
-- 然后获取元表来设置
getmetatable(mytable)
```

## 元方法

- `__index`方法
    * 当访问table时，一个键没有值，则寻找`__index`中对应键的值或调用`__index`函数
    * 如果`__index`是table，返回对应键的值，否则nil
    * 如果`__index`是函数，返回执行结果
        + 可以有两个参数(table, key)
- `__newindex`方法
    * 当给table中一个缺少的索引赋值时，会执行`__newindex`方法(如果存在的话)，**而不给这个索引赋值**
    * 如果`__newindex`是table，则最newindex中的table对应的索引赋值，而不对原table赋值
    * 如果`__newindex`是方法，则调用之
        + 可以有三个参数(table, key, value)
- `__add`...等元方法







