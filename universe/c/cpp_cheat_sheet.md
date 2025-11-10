---
title: CPP cheat sheet
author: 66RING
date: 2000-01-01
tags: 
- cpp
mathjax: true
---

# 小抄

> stack

## 妙用structure bind

- case 1: 使用pair等时用`p.first`, `p.second`有很差的可读性。可以使用`auto [name, age]`
- case 2: 除了pair, 结构体也可以使用bind。绑定的顺序和结构体成员顺序有关

```cpp
struct Person {
    string name;
    int age;
};

Person get_person();

auto [name, age] = get_person();
```

- case 3: 当有多个输出参数时`func(int n, int& out1, int& out2)`，可以利用bind的方法放回一个结构体然后分解

## 右值引用

> 所有权语义
>
> 尽可能少复制的情况下把数据存到内存

- 直接从语言层面理解，就是rust的move
- move后所有权转移, 不应该再使用了 => 将亡值
    - 软约束，继续使用可能会出现未定义行为
- 右值引用作为参数 => 该函数将夺取该参数的所有权


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


## push_back 和 emplace_back的区别

- `push_back`: 新建然后再移动或拷贝
- `emplace_back`: **可以**在容器开辟的空间内原地创建

之所以说的"可以"是因为使用不当还是会触发移动/拷贝的。用法应该是传给`emplace_back`的参数的构造函数的参数。

```cpp
#include <iostream>
#include <stdio.h>
#include <unistd.h>
#include <vector>

using namespace std;
class S {
public:
  int n;
  S(int n) : n(n) { cout << "Cons\n"; }
  S(S &&s) : n(s.n) { cout << "move\n"; }
  S(const S &s) : n(s.n) { cout << "copy\n"; }
};

int main() {
  vector<S> v;
  v.reserve(20);
  v.emplace_back(10);       // 构造
  v.emplace_back(S(10));    // 构造 + move
}
```


## 多线程和锁

```cpp
#include <thread>
#include <mutex>

void func(int &x) {}

class ThreadObj {
public:
  void operator()(int x) {}
};

int main() {
  std::thread th1(func, arg);  // 通过函数启动的
  std::thread th2(ThreadObj, arg);  // 通过类对象启动的
  std::thread th3(lambda, arg);  // 通过匿名函数启动的
  std::this_thread::get_id()
  std::this_thread::()
  this_thread::sleep_for(std::chrono::seconds(1));
  th1.join()// 等待完成

  // mutex:
  std::mutex mymutex;
  mymutex.lock();
  mymutex.unlock();

  // lock_guard
  std::mutex mymutex;
  std::lock_guard<std::mutex>(mymutex); // 已废弃, 使用scoped_lock
  std::scoped_lock<std::mutex> lock(mymutex);
  
  // 显式引用/线程引用参数的传递(e.g. 锁)
  std::ref();

  int a;
  std::thread th4(func, std::ref(a));
}
```

## 文件流

```cpp
#include <iostream>
#include <fstream>
using namespace std;
int main() {
  ofstream file;
  char data[100];
  file.open("filename");
  cin.getline(data, 100);
  cin.ignore(); // 跳过直到下一个\n

  // seek
  file.seekp(0, ios::beg);
  file.close();

  // 只读
  ifstream rfile;
  rfile.open("./a.txt");
  rfile >> data;
  cout << data << endl;
  rfile.close();
  return 0;
}
```

## misc

```cpp
struct sdshdr {
    long len;
    long free;
    char buf[]; //不占空间
};
```

## typename

为什么叫typename, 因为在模板中, 泛型T可能是类型, 也可能是变量。使用模板时提取出来的是类型还是变量存在二意性，所以需要用typename来明确指出是类型。

```cpp
template <typename T, int arg>
struct Type<T> {
    using type = T;
    int arg = arg;
};

template <typename T>
void show(typename Type<T>::type a) {
    std::cout << a << std::endl;
}
```

`typename Type<T>::type`说明`Type<T>::type`是一个类型，而不是一个变量。

为什么template中已经用了typename修饰T了后面没能推导出type是类型呢？因为内部type需要具体特化时才能分析到。






