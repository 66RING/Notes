---
title: c++进阶
date: 2020-5-15
tags: c++
---


## 叫什么呢

### 关键字decltype(c++11)

关键字decltype(x)用于自动检测x的类型，并作为关键字。使用方法如下

``` c++
int x;
decltype(x) y;   // 声明y，其中y的类型取决于x
decltype(x+y) xpy;   


// 格式如下
decltype(expression) var;  // expression可以是函数调用
```


#### 另一种函数声明语法(c++11后置返回类型)

考虑下面的情形：

``` c++
template<classs T1, class T2>
?type? gf(T1 x, T2 y){
    ...
    return x+y;
}
那函数的返回类型是什么呢？
```

显然只用decltype是解决不了问题了。使用新增的语法可编写成这样：

``` c++
template<classs T1, class T2>
auto gf(T1 x, T2 y) -> decltype(x+y)
{
    ...
    return x+y;
}
```




