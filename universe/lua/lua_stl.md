---
title: Lua标准库工具
date: 2020-10-06
tags: lua
---

## base

### \_G

`_G`是lua中的一个全局变量，其中保存了lua语言中几乎所有的全局函数和变量

如果全局变量(或函数)存在的话，可以使用`_G["VAR"]`获取到全局变量。但不能获取到局部变量。可以通过`_G["VAR"]=local_var`将局部变量转变为全局变量


## string

- `string.gmatch(s, pattern)`
    * 返回一个迭代器函数，每次调用这个函数，返回一个在字符串`s`找到的下一个pattern描述的子串。没找到返回`nil`
    * 类型类似正则表达式(或者就是)
        + `%a`查找一个字母
        + `%w`查找一个数字或字母
        + `%d`查找一个数字
        + `(%w+)=(%w+)`，`()`分组，返回成对的，`+`找一个及以上
- `string.find(str, substr, [init, [end]])`
    * 从字符串中搜索子串，返回位置索引，不存在返回nil
- `string.gsub(mainString,findString,replaceString,num)`
    * 替换，执行num次


## table
