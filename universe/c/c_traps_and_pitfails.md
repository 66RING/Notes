---
title: C陷阱和缺陷
date: 2020-08-07
tags: 
- C
- traps
mathjax: true
---

> 对C陷阱和缺陷一书的记录

`(*(void(*)())0)()`

- tips
	* 常数 != 指针，要先将常数转换为指针`(void(*)())`
	* 函数签名:`type NAME(args)()`，故除掉NAME部分表示它的类型

# 添加语句缺陷

else与最近的if进行配对。如：

```
if(A) foo();
if(B) bar();
else done();
```

else会与`if(B)`配对，你可能会觉很莫名奇妙。因为一般作用范围会用花括号`{}`界定，不会出现这种"意外"。但如果你在开发大型项目，或者宏定义中使用了if呢？


