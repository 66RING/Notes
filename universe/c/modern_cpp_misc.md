---
title: 现代cpp杂项
author: 66RING
date: 2000-01-01
tags: 
- cpp
mathjax: true
---

[TOC]

## cpp单例模式

static标记, 返回引用。static标记后构造函数就只会触发一次。

```cpp
class A {
public:
  static A &getinstance() {
    static A a;
    return a;
  }
};
```


## constexpr标注

编译期就能确定(编译器执行), 从而不需要运行时, 从而优化执行效率。e.g. 一些递归constexpr的计算: 阶乘

声明时constexpr标记表示可以在编译期确定, 不过也可以用在运行时。调用时使用`constexpr`表示显式使用编译期执行, 如果不能编译器会报错。否则就是自动选择编译期还是运行时运行。 如果调用要运行时才能决定, 如`frac(n)`等待用户输入一个n就"自动取消constexpr"

```cpp
constexpr int frac(int n) {
    if (n == 1) {
        return 1;
    } else {
        return n * frac(n-1);
    }
}

int main() {
    constexpr int result = frac(5);
}
```

前后对比: `objdump -dC <binary>`


## explicit

对一个构造函数进行`explicit`修饰, 可以防止隐式调用一个参数的构造函数

```cpp
class A {
 public:
  A(int x);
  // explicit A(int x);
};

doSomething(28);  // 产生了隐式临时对象
doSomething(A(28));  // 产生了显示临时对象
```

## enum class: T

旧enum存在许多问题:

1. 隐式转换成整型
2. 无法自定义类型
3. 存在作用域问题, 可以直接通过enum的成员名访问成员
4. 取决于编译

`enum class Name {}`语法解决了旧`enum`的问题

1. 不再隐式转换, 可以手动强转
2. 指定底层数据类型: `enum class Name: T {}`
3. 作用域访问成员需要使用域运算符

