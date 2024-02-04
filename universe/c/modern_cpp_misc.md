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

## X Macro

> [知乎教程](https://zhuanlan.zhihu.com/p/521073931)

使用宏技巧自动生成模式, 如定义变量, switch case等模式

1. 小范围内定义宏, 然后释放(undef)
2. 本质就是利用宏函数的一些功能来实现诸如字符串拼接, 转字面值, 转"值"等操作, 如
    - `#define X_MACRO(x) x,`, 原样输出并添加一个都好
    - `#define X_MACRO(x) #x`, 输出字符串字面值"x"

```cpp
#define XMACROS_TABLE(f) \
	f(trace) \
    f(debug) \
    f(info) \
    f(critical) \
    f(warn) \
    f(error) \
    f(fatal)

enum class log_level : std::uint8_t {
// X Macro原样返回name, 快速完成定义
#define _FUNCTION(name) name,
    XMACROS_TABLE(_FUNCTION)
#undef _FUNCTION
};

inline std::string log_level_name(log_level lev) {
    switch (lev) {
// X Macro批量生成 case log_level::name: return #name
#define _FUNCTION(name) case log_level::name: return #name;
    XMACROS_TABLE(_FUNCTION)
#undef _FUNCTION
    }
    return "unknown";
}
```

## 类型擦除

> 可以实现其他语言中interface之类的特性

我们知道虚基类的作用是提供一个统一的接口。即如果将子类赋值到基类则调用虚函数饰会自动调用对应的实现, 从而实现统一的管理。类型擦除就是利用虚基类能提供统一接口的这个原理, 将虚基类的自动"分发"封装在一个类中, 该类再对外提供统一的模板和接口, 从而隐藏虚基类的调用。

如下面这个`Function<void(int)>`可以接收任何实现了`operator()()`方法的对象, 包括结构体的, 函数指针和lambda。

```cpp
// NOTE: 关闭默认特化, 必须使用函数形式特化
// 为什么不能直接false, 因为要依赖模板FnSig做惰性编译
template <class FnSig>
struct Function {
  static_assert(!std::is_same_v<FnSig, FnSig>, "not a valid function signature");
};

template <class Ret, class ...Args>
// 特化: 使用时用这种格式, Function<void(int)>
// NOTE: 偏特化: 如果存在这种形式就不会跳到上面FnSig处了
struct Function<Ret(Args...)> {
  struct FuncBase {
	virtual Ret call(Args ...args) = 0;
	virtual ~FuncBase() = default;
  };

  template <class F>
  struct FuncImpl : FuncBase {
	F f;

	FuncImpl(F f) : f(std::move(f)) {}

	// 把FuncBase的call覆盖掉, 就是调用传入的函数F
	virtual Ret call(Args ...args) override {
	  return std::invoke(f, std::forward<Args>(args)...);
	}
  };

  std::shared_ptr<FuncBase> m_base;

  Function() = default;

  // 构造函数, 会实例化多次, 但都用统一的虚接口
  template <class F, class = std::enable_if_t<std::is_invocable_r_v<Ret, F &, Args...>>>
  Function(F f): m_base(std::make_shared<FuncImpl<F>>(std::move(f))) {}

  Ret operator()(Args ...args) const {
	if (!m_base) [[unlikely]]
	  throw std::runtime_error("function uninitialized");
	// 虚接口实例化多次, 虚函数自动调用对应实现
	return m_base->call(std::forward<Args>(args)...);
  }
};
```

主要有三个部分组成: "接口类", "impl类", "基类"。接口类将后两者封装, 内含一个基类成员以获取统一接口, impl类继承自基类那么就可以使用模板自动构造类。而使用时我们只需要关注接口类的使用。


## decltype

`decltype(e)`, 总是以一个普通表达式作为参数, 做编译期类型推导

```cpp
int i = 4;
decltype(i) a;
```

泛型编程中结合auto来追踪变量返回值:

```cpp
template <typename _Tx, typename _Ty>
auto multiply(_Tx x, _Ty y)->decltype(_Tx*_Ty)
{
    return x*y;
}
```

## static switch

利用泛型自动生成不同数据规模的实现, 本质就是枚举所有可能的数据规模。然后传入一个闭包来限制作用域。

- `__VA_ARGS__`用于展开宏中的可变参数, 因为传入是的一个闭包所以`__VA_ARGS__()`手动调用
- 枚举所有可能并生成所需的静态变量, 以完成泛型展开

```cpp
// https://github.com/NVIDIA/DALI/blob/main/include/dali/core/static_switch.h
// and https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/Dispatch.h

#define FWD_HEADDIM_SWITCH(HEADDIM, ...)   \
  [&] {                                    \
    if (HEADDIM <= 32) {                   \
      constexpr static int kHeadDim = 32;  \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 64) {            \
      constexpr static int kHeadDim = 64;  \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 96) {            \
      constexpr static int kHeadDim = 96;  \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 128) {           \
      constexpr static int kHeadDim = 128; \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 160) {           \
      constexpr static int kHeadDim = 160; \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 192) {           \
      constexpr static int kHeadDim = 192; \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 224) {           \
      constexpr static int kHeadDim = 224; \
      return __VA_ARGS__();                \
    } else if (HEADDIM <= 256) {           \
      constexpr static int kHeadDim = 256; \
      return __VA_ARGS__();                \
    }                                      \
  }()

FWD_HEADDIM_SWITCH(dim, [&] {});
```

## 模板函数的header分离

> 模板函数的"声明"并不是常规意义是声明, 只**是用来生成声明的蓝图**

如下场景会发现找不到symbol。即将模板函数的声明写在header将"定义"分开写在一个cpp文件。

这是因为cpp文件中的"定义", **对于模板函数来说并不是常规意义定义**。模板函数是蓝图: 在使用到时才自动展开。所以在编译`add.cpp`时没有任何使用蓝图就不会生成任何具体的声明。当编译`main.cpp`时就没有发现任何声明。

```cpp
// add.h
#pragma once
template <typename T> T add(T a, T b);

// add.cpp
#include "add.h"
template <typename T> T add(T a, T b) { return a + b; }

// main.cpp
#include "add.h"
#include <iostream>

using namespace std;

int main() {

  cout << add(1, 2);
  return 0;
}
```

改进方法: 要么都写在header文件中, 要在模板函数手动特化, 如

```cpp
// add.cpp
#include "add.h"
template <typename T> T add(T a, T b) { return a + b; }
template int add(int, int);
template<> float add<float>(float, float); // 或者
```







