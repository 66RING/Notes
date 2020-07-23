---
title: lua快速上手
date: 2020-7-23
tags: lua
---

## 概述

类似python等脚本语言，在循环、方法、条件等语句需要使用end表示结束，因此不像python那样依靠缩进。

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


### 条件
``` lua
if a>b then
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


### **面向对象**

lua本身没有面向对象，但是可以通过某种方式实现

- **使用表的形式** (类似golang)
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
        p:hello()   -- 默认传入自己
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
- **使用函数闭包的形式面向对象** ，可以实现私有化，但是表的方式更块些
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








