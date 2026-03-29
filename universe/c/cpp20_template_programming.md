---
title: c++20模板元编程
author: 66RING
date: 2025-12-01
tags: 
- cpp
- modern cpp
mathjax: true
---

# c++20模板元编程

## tips

- 主模板 + 偏特化
    * AKA 定义默认行为
- 类型擦除
- 奇异递归
- 模板表达式
- 标签派发
- **静态面向"对象"**
    * **静态多态**:
        - 奇异递归
    * **鸭子类型**: e.g. golang中的interface
        - 混入(mixin): 静态检查
        - 类型擦除, 没有多态的通用处理(但有相同的interface)


## 变参模板

- `...`运算符表示解包, 在什么后面就是对什么的解包
    * `template<typename... T>func(T... args)`: 对形参T解包, 其中`T...`称为**形参包**
    * `call_func(args...)`: 对实参args解包


- 变参目标可以匹配多个形参包: **贪婪匹配**: 直到形参包无法匹配是才推导下一个

```cpp
template<typename... Ts, typename... Us>
constexpr auto multipacks(Ts... a0, Us... a1) {
  static_assert(sizeof...(a0) == sizeof...(a1));
}

multipacks<int>(1, 2, 3, 4, 5, 6); // fail: int贪婪分配给Ts
multipacks<int, int, int>(1, 2, 3, 4, 5, 6); // ok: int,int,int贪婪分配给Ts
```

## 变参类

e.g. **tuple**

```cpp
template<typename T, typename... Ts>
struct tuple {
  tuple(const T& t, const Ts&... rest): first_(t), rest_(rest...) {};
  constexpr int size() { return 1 + rest_.size(); }

  T first_;
  tuple<Ts...> rest_;
};

template<typename T>
struct tuple {
  tuple(const T& t): first_(t) {};
  constexpr int size() { return 1; }

  T first_;
};
```

e.g. `get<N>`

```cpp
template<int N, typename T, typename... Ts>
constexpr T& get(tuple<T, Ts...> &t) {
  if constexpr (N > 0) {
    return get<N-1>(t.rest_);
  } else {
    return t.first_;
  }
}
```

## 折叠表达式

e.g.

```cpp
template<typename... Ts>
int sum(Ts... args) {
    return (... + args);
}
```

4种折叠: `...`在pack的左边还是右边

- 无init: 一元折叠
    * 一元右折叠: `pack op ...`
        + (arg1 op (arg2 op (arg3 op argN)))
        + 越靠右结合律越高
    * 一元左折叠: `... op pack`
        + (((arg1 op arg2) op arg3) op argN)
        + 越靠左结合律越高
- 有init: 二元折叠, **结合规律同理一元, 区别在于init要cat在左边还是右边**, 拼接init后做一元折叠
    * 二元右折叠: `pack op ... op init`
        + (arg1 op (arg2 op (arg3 op (argN op init)))
    * 二元左折叠: `init op ... op pack`
        + ((((init op arg1) op arg2) op arg3) op argN)

## 移动语义

> in short. 左值: "有存储位置", 右值: "临时的, 没有存储位置, 在调用之外不存在"

完美转发: 因为左值引用的**实参本身也是一个左值**, 所以对这个实参进一步传递时只会调用左值的重载。

```cpp
void g(int& v) { std::cout << "g(int&)\n"; }
void g(int&& v) { std::cout << "g(int&&)\n"; }

void h(int& v) { g(v); }  // v是左值
void h(int&& v) { g(v); } // v是左值

// NOTE: 改进
void h(int& v) { g(std::forward<int&>(v)); }
void h(int&& v) { g(std::forward<int&&>(v)); }


int x{1};
h(x);      // print -> g(int&)
h(int{1}); // print -> g(int&)
```


## declval

`std::declval<T>()`生成一个类型T的**值(对象)**, 且: 是右值引用 + 无需调用构造函数。

e.g. 为了解决如下问题: 需要值(对象)来做推导, 但是没有默认构造

```cpp
template <typename T, typename U>
struct composition {
    // 需要默认构造
    using type = decltype(T{} + U{});
    // 不需要默认构造
    using type = decltype(std::declval<T>(); + std::declval<U>());
};
```

## 条件编译

**主模板 + 偏特化**: 偏特化失败就会落到主模板。e.g.

```cpp
template<typename T>
struct is_floatpoint {
    static const bool value = false;
};

template<>
struct is_floatpoint<float> {
    static const bool value = true;
};

template<>
struct is_floatpoint<double> {
    static const bool value = true;
};

template<>
struct is_floatpoint<long double> {
    static const bool value = true;
};
```

往往搭配其他模板使用, e.g. 一个true特化的, 一个false特化的。


## SFINAE

> Substitution failure is not an error(替换失败不是错误)
>
> AKA模板的匹配机制: 不匹配(替换失败)就尝试其他的替换

"自动条件编译": 哪个匹配用哪个

TODO: review


## 条件编译

SFINAE会调一个匹配的实现。当匹配到`B=false`的实现时, 在使用时就会编译报错。

```cpp
template <bool B, typename T = void>
struct enable_if {};

template <typename T>
struct enable_if<true, T> { using type = T; };
```

some how等价于

```cpp
if constexpr (true) {
} else {
}
```

## type trait

> 类型特征
>
> "用模板的方式加各种限定" + 别名/工具包


## 概念和 requires

> 背景: 只查看声明不检查主体无法知道确定的功能(比如可能被奇怪的重载了)
>
> 目的: 增加可读性

requires用法: "类似enable_if, 但是报错更友好"

```cpp
template <typename T>
requires std::is_arithmetic_v<T>
T mul(T const a, T const b) {
    return a * b;
}
```

concept用法: "类似using, 是typename + requires的语法糖"

```cpp
template <typename T>
concept arithmetic = std::is_arithmetic_v<T>;

template <arithmetic T> // <=================
T mul(T const a, T const b) {
    return a * b;
}

template <arithmetic T> // <=================
T add(T const a, T const b) {
    return a + b;
}
```

### 简单要求

> 根据表达式正确的确定true/false

编译器会检查是否实现了`+`

```cpp
template <typename T>
concept addable = requires(T a, T b) {
    a + b;
    a - b; // ...
    func(a, b);
};
```

### 类型要求

> 约束`<typename T>`的T的简单要求

e.g. 一个类型必须包含内部类型`key_type`和`value_type`

```cpp
template <typename T>
concept KVP = requires { // 没有形参可以省略不写
    typename T::key_type;
    typename T::value_type;
};
```

用法同理requires

```cpp
template <KVP T>
...

// 或者
template <typename T>
requires KVP<T>
...
```

### 符合要求

> 要求含有某个方法和返回类型

```cpp
template <typename T>
concept timer = requires(T t) {
    {t.start()} -> std::same_as<void>;
};
```

### 嵌套要求和组合要求


```cpp
template <typename T>
concept timer = requires(T t) {
    requires sizeof(T) > 1;
    requires are_same_v<T>;
};


template <typename T>
requires A && B || C
...
```

## 静态多态

> 模板自动生成多态的具体实现

e.g. `t.value++`有具体的类型实现(具有多态性)

```cpp
struct attack {
    int value;
};
struct defent {
    int value;
};

template <typename T>
void increment(T& t) { t.value++; }
increment(attack {10});
increment(defent {10});
```

## 奇异递归模板模式

> CRTP
>
> AKA静态多态 + 面向对象, 可以节省查虚表的开销

模式如下:

1. 有一个定义接口的基类类模板(静态)
2. 派生类本身是基类模板的实参
3. 基类函数调用派生类函数

tips: 因为都实现了统一的接口, 所以通过基类方式统一调用就行。模板负责生成不同的重载。

```cpp
template <typename T>
struct game_unit {
  void attack() {
    static_cast<T*>(this)->do_attack();
  };
};

struct knight: game_unit<knight> { // NOTE: 类是自身的模板
  void do_attack() {}
};

struct mage: game_unit<mage> { // NOTE: 类是自身的模板
  void do_attack() {}
};

knight k0;
mage k1;
k0.attack();
k1.attack();
```

## 混入

> mixin
>
> 编译期功能组合, 编译器检查有没有实现所需的功能
>
> AKA鸭子类型, 只要有某个interface就行

e.g. 可以检查出T有没有都实现所需的功能

```cpp
many_funcs<T> k;
k.func0();
k.func1();
...
```

## 类型擦除

> 通用方式处理**不一定相关**的类型。e.g. 没有继承/多态关系也能通用处理

- 无继承/多态关系的通用处理, 但是**鸭子类型**
    * `void*` + 类型转换是一种方法, 但类型不安全
- 思想: **创建一个通用的间接类, 该类内具体的对象实例由模板生成**
    1. A和B没有直接的关系, 但有相同的接口
    2. 为了不改变A, B之间的关系, 可以试想一个间接的连接
        - e.g. `concept_unit_for_A`, `concept_unit_for_B`, 他们都继承了unit, **有相同的接口, 但内部实例化的对象分别是A, B**
    3. 用间接类做通用操作即可
- 套路
    * 1. 定义一个concept, 表示共有的接口
        + `struct concept{}`
    * 2. 模板定义一个间接类, 间接类都继承了concept
        + `template<class T> struct _base: concept{}`
    * 3. 使用时对外暴露操作的其实是间接类, 模板规划会生成对应的示例
        + `_base.inferface()`
    * 4. fwd语义 + 智能指针

e.g.

```cpp
struct unit {
  template <typename T>
  unit(T&& obj): 
    unit_(std::make_shared<unit_model<T>>(std::forward<T>(obj)))
  {}

  void attack() {
    unit_->attack();
  }

  // 接口定义
  struct unit_concept{
    virtual void attack() = 0;
    virtual ~unit_concept() = default;
  };

  // 间接类(间接存在明显的多态关系, 都继承自concept)
  // 模板自动生成具体类型实例
  template <typename T>
  struct unit_model: public unit_concept {
    unit_model(T& unit): t_(unit) {}
    void attack() override { t_.attack(); }
  private:
    T& t_;
  };

private:
  std::shared_ptr<unit_concept> unit_;
};
```

## 标签派发

> 编译期函数选择(或者重载选择). e.g. 容器类型

- wrapper函数内部通过trait决定调用哪种实现
- 替代方案
    1. `if constexpr`
    2. concept约束

```cpp
template <typename Iter, typename Distance>
void advance(Iter& it, Distance n) {
  // advance有多种函数重载
  detail::advance(it, n,
    typename std::iterator_traits<Iter>::iterator_category{})
}
```





