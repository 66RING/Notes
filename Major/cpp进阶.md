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

#### 初始化结构体

``` 
struct ListNode {
    int val;
    ListNode *next;
    ListNode(int x) : val(x), next(NULL) {}
};
ListNode* ls=new ListNode(0);
// ls->val=0;ls->next=NULL;
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


## 对象和类

### 构造函数和析构函数

类需要构造函数来创建类对象，不能像下面的那样初始化对象，因为数据部分的访问状态是私有的，这意味着程序不能直接访问数据乘员。

``` c
Person one = {"Bob", 23};  //编译错误
```

- 使用构造函数的两种方法：
    - `Person one = Person("Bob", 23);`
    - `Person one("Bob", 23);`
- 将构造函数与new一起使用
    - `Person *one = new Person("Bob", 23);`

- 析构函数
    - `~Person(){}`
    - 构造函数创建对象后，程序负责跟踪该对象，直到过期。过期时自动调用析构函数
    - 一般用于删除分配了的资源
- 默认构造函数
    - `Person(){}`
- 列表初始化
    - C++11中，可将列表初始化语法用于类中(构造函数)
    - `Person one = {"Bob", 23};`
    - `Person one{"Bob", 23};`
    - `Person one{};` 调用默认构造函数
        - 不同于`Person one()`; 这是一个返回Person的函数
- const成员函数
    - `const Person one("Bob", 23);`  
    `one.show()`，这样是不行的因为不能保证show方法不会修改对象
    - 除非show方法为：`void show() const;`，这就是声明const成员函数的方法
    - 因此只要类方法不修改调用对象，就应该将其声明为const



