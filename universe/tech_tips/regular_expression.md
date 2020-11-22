---
title: 正则表达式基础语法
date: 2020-1-23
tags: Regex, tech, tool
---

Base/Extended Regex

- `.`
    - 匹配任何一个字符
- `*`
    - (贪婪)匹配任何数量(包括0)的前面的内容，贪婪匹配
- `+`
    - 匹配一个或多个的前面的内容，非贪婪匹配
- `-`
    - 匹配零个或多个的前面的内容，非贪婪匹配
- `{1}`
    * 重复次数1
- `\S`
    - 任何非空白字符
- `\s`
    - 任何空白字符
- `?`
    - 前面内容是可选的
- `\`
    - 有时可以起到转意的作用，`\S`等就是例外
- `[0-9]`
    - 任何0到9的数字，范围可变但要按顺序
- `[a-z]`
    - 任何a到z的小写字符，范围可变但要按顺序
- `[A-Z]`
    - 任何A到Z的大写字符，范围可变但要按顺序
- `[A-Za-z]`
    - 任何A到Z的字符，范围可变但要按顺序
- 锚定
    * `$`
        + 以前面的内容结束
    * `^`
        + 以前面的内容开头
    * `\< ,\b`
        + 锚定单词首部
    * `\> ,\b`
        + 锚定单词结尾
- `|`
    * or，整个左边或整个右边如`C|cat`表示C或cat
    * 也可使用分组改变执行顺序`(C|c)at`表示Cat或cat