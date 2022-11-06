---
title: CPP中的左值和右值概念
author: 66RING
date: 2022-10-17
tags: 
- cpp
mathjax: true
---

# CPP中的左值和右值概念

## Overview

- 语法上能取地址就是左值
- 有名引用本身是左值, 无名右值引用(`std::move()`的返回)是右值
- `const`左值引用(`const T& val = 6`)引用右值, 可以编译通过。 因为与右值不指代对象, 没有"存储"的语义没有冲突
- 性能上没差别, 引用都可以避免拷贝
- 左值引用可以使用`const T&`引用右值
- 右值引用可以使用`std::move()`引用左值


## 发展史

原始版本是说赋值号的左右边。

- CPL
	* 赋值号的左边应该是个地址, 这样才能存储
- C
	* lvalue
	* other: rvalue和function
- CPP
	* 98: lvalue(包含函数), rvalue
	* 11: 开始复杂


## C中的左右值

> 是否真正指代一个对象, 都是语法上的东西, 毕竟临时对象也是存在栈上的

**能取地址就是左值**

- lvalue: `locator-value`
	* 指代对象
	* 存储地址
	* 当然也可以当右值用
	* locator != 可赋值
		+ 如`char a[2]; a = "hi";`
- rvalue
	* 不指代对象(虽然一定存在内存某处), **只有值是有意义的**

举个例子, 一个返回对象的函数`T f()`。你可以用来赋值新结构体`T t = f()`, 但不能`f() = xxx`, `f()`就是右值。

- 左值引用, 右值引用: 指向左值/右值的引用
    * 左值引用无法指向右值, 因为右值并不指代对象, **不能"存储"/修改**
        + 可以使用const左值引用来引用右值
- 右值引用
    * 符号: `&&`
    * 专门引用右值, **不能引用左值**, 除非：
        + 使用`std::move`让右值引用可以引用左值, 虽然叫`move`但至少"别名"
    * **利用右值引用可以做到修改右值的效果**
    * **本质上右值引用是将一个右值提升为左值**, 然后利用`std::move`引用左值

```cpp
int&& ref = 4;
ref = 6; // 可以修改右值引用
int a = 5;
int &&ref1 = a; // ❌右值引用不可用指向左值
int &&ref2 = std::move(a);  // std::move可以让右值引用引用左值
```


## 有名引用本身是左值

有名引用本身是左值, 但右值引用(引用本身)可以是右值也可以是左值

```cpp
void f(int &&rvalue) {
    rvalue = 1;
}

int main() {
    int a = 5;
    int &lvalue = a;
    int &&rvalue = std::move(a);

    f(a);       // ❌
    f(lvalue);  // ❌
    f(rvalue);  // ❌, 都需要std::move()
}
```

为什么说右值引用(本身)即可以是右值也可以是右值?

因为`std::move()`返回的是一个右值引用, 而上面的例子说明了右值引用是个左值, 而`int &&`右边又必须是个右值, 所以这种情况下`std::move`返回的右值引用是个右值。

结论: 有名右值引用本身是左值, 无名右值引用本身是右值


## 完美转发

本质上也是做类型转换, `std::move()`转换出右值, 而`std::forward<T>()`即能转换出右值也能转换出左值, 有泛型`T`决定。

前面[有名引用本身是左值](#有名引用本身是左值)说过有名引用都是左值。那我们拿到一个引用后(通过函数传递或临时变量等方式)要怎么把这个左值传递下去? 就可以使用`std::forward<T>()`了


## 移动构造

比如容器扩容

```cpp
class SuperClass{
    public:
        int * p;
        SuperClass(int _i):p(new int(_i)){
            //构造函数
        }
        SuperClass(const & SuperClass another):p(new int(*another.p)){
            //拷贝构造函数
            //因为是const，无法接管,只能深拷贝
            //如果把p接管，无法将another的p置空（const修饰），这样在析构会导致p被释放两次而报错
        }
        SuperClass(SuperClass && another):p(another.p) noexcept{
            //移动构造函数，将another的成员聚合到当前对象（接管过来）
            //然后将原有对象的成员置空（防止析构delete）
            another.p = nullptr;
        }
        ~SuperClass(){
            delete p;
        }
};
```


