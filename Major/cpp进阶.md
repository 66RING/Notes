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


### 说明符和限定符

- auto
- register
- static
- extern
    - 指出是对外部变量(相对当前代码块的外部)的引用
- thread\_local(C++11)
- mutable
    - 即使结构变量为const，mutable的成员也是可以被修改的
- volatile
    - 表明，即使程序代码没有对内存单元进行修改，其值也可能发生改变


### 使用new运算符初始化

如果要为内置的标量类型分配存储空间并初始化，可以在类型名后面加上初始化值，并将其用括号括起：

``` c++
double* pd = new double (99.99)  // *pd=99.99
```

要初始化常规结构或数组，需要使用打括号

``` c++
struct where{double x; double y; double z;}
where* one = new where{2.3, 3.2, 6.4}
int* ar = new int[4]{2, 3, 3, 4}
```

#### 定位new运算符

通常，new负责在堆中找到一个足以满足要求的内存块。new也可以指定要使用的位置：

``` c++
char buffer[20];

int p1 = new (buffer) int; //在buffer中找空间
```


### 名称空间

当项目很大的时候可能会发出重名的现象，这时可以使用名称空间进行区分。

C++新增了这样一种功能，即通过定义一种新的声明区域来创建命名的名称空间：

``` c++
namespace Jack{
    double pail;
    void fetch();
}
```

using声明将特定的名称添加到它所属的声明区域中：

``` c++
chat fetch;
int main(){
    using Jack::fetch();
    double fetch;    // 失败，因为已经声明了fetch
    cin >> fetch;    // Jack的fetch
    cing >> ::fetch; // 全局的fetch
}
```

using声明使一个名称可用，而using编译指令使所有的名称都可用。using编译指令使用`using namespace`关键字。

`
using namespace Jack;
`

可以将名称空间声明进行嵌套

``` c++
namespace Bob{
    namespace Bill{
        int age;
    }
    string name;
}
```

也可以在名称空间中使用using编译指令和using声明：

``` 
namespace myth{
    using Jack::fetch;
    using namespace Bob;
}
```





