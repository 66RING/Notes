---
title: CPP cheat sheet
author: 66RING
date: 2000-01-01
tags: 
- cpp
mathjax: true
---

# 小抄

## 移动构造

使用右值引用创建移动构造器。然后传入右值: `std::move(lvalue)`或者直接匿名变量

```cpp
class A {
    A(A &&a) {
        std::cout << "move\n";
    }
}

int main() {
    vecotr<A> v;
    // 使用匿名遍历
    v.push_back(A());
    A a;
    // 使用move
    v.push_back(std::move(a));
}
```

## 智能指针

> 自动资源管理

- 使用new创建: `std::unique_ptr<int[]> ptr(new int[10]);`
- 使用api创建: `std::unique_ptr<int[]> ptr = std::make_unique<int[]>(10);`
- `unique_ptr`, 所有权唯一
- `share_ptr`, 所有权不唯一, 当所有引用都释放时才自动释放


## 类型转换

- `reinterpreter_cast<T>(value)`, 等价于c中的强转, 不存在运行时检测, cpp中出现是为了统一接口风格
    * 问题: 你知道转换是安全的才能安全, 否则UB, e.g. 整型转指针, 基类转派生
- `const_cast<T*/T&>(ptr/ref)`
    * 去除const属性, e.g. 去除常成员函数this指针的const限制
    * 只能对指针/引用转换
- `static_cast<T>(value)`
    * 明确的隐式转换, 明确是指给程序员明确; 隐式是指使用隐式转换规则, 而不是直接内存截断
- `dynamic_cast<T*/T&>(ptr/ref)`
    * 带虚函数的子类与父类的指针/引用转换
    * 父类转子类不安全, 但是没有提示, 所以有了`dynamic_cast`就做运行时检测, 不安全时返回nullptr
    * 情景: 子类转成了父类, 然后再用这个父类转回子类
    * 必须要有虚函数, 因为它底层的RTTI(运行时类型检测)依赖虚函数




