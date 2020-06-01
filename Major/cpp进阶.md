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


## 类和对象

### 使用类

#### 重载运算符

- 格式：`operator op(argument-list)`
    - 如`operator +()、operator *()`
    - 不能是一个虚构的符号

假设有个一个Bob类，并为它定义了一个`operator +()`成员函数，以重载+运算符。A、B和C都是Bob的对象。便可以编写如下等式：
- `C = A + B`
- 编译器发现操作数是Bob对象，因此使用相应的运算符函数代替上述运算符
    - `C = A.operator+(B)`
    - 这说明了运算符的原理

重载限制：
- 重载的运算符必须至少有一个操作数是用户自定义的类型，防止用户为标准类型重载运算符
- 使用运算符不能违反运算符原来的语句规则
- 不能修改运算符的优先级
- 不能创造新的运算符


### 友元

C++提供形式的访问权限，：友元。友元有3中：
- 友元函数
- 友元类
- 友元成员函数


#### 友元成员函数

友元成员函数可以访问类内的私有成员，通过让函数成为类的友元，可以赋予该函数与类的成员函数相同的访问权限。

考虑下面的情形：设A、B都是Bob类的对象，并且Bob类重载了\*运算符。则`A = B * 2.5`可行，但`A = 2.5 * B`就会出现问题。原因在于，类内对运算符重载，隐式调用对象`A = B.operator*(2.5)`，而换位置后`2.5`对象并没有重载运算符。那难道要重载`double`类的运算符吗？那将会造成很大混乱。

还有种重载运算符的方法：使用非成员函数。如`Time operator*(double m, const Time& t)`运算符左边对应第一个参数，右边对应第二个参数。

那如果Time类中有私有数据呢，非成员函数怎么访问？这时就需要友元了。

- 创建友元
    - 将原型放在类声明中，并加上`friend`关键字
        - 该原型意味着下面两点
        - 函数是在类中声明的，但不是类成员函数，因此不能用成员运算符来调用
        - 不是成员函数，但数据的访问权限相同
- 编写函数定义
    - 因为不是成员函数，所以不能用`Class::`限定符，就像声明普通函数一样即可


### 静态成员

静态类成员有一个特点：无论创建多少对象，程序都只创建一个静态类变量副本。即类的所有对象共享同一个静态成员。

不能在类声明中初始化静态成员变量，因为声明描述如何分配内存，但不执行分配。需要在声明之外使用单独的语句进行初始化， **因为静态类是单独储存的，而不是对象的组成部分** 。


### 特殊成员函数

- C++自动提供下面这些成员函数(如果没有定义)
    - 默认构造函数
    - 默认析构函数
    - 复制构造函数
        - 它接受一个指向类对象的常量引用作为参数`Class_name(const Class_name&);`
        - 何时调用
            - 用类对象生成另一个对象`Time a(b);`
            - `Time a = b;`
        - 复制是按值复制的，也就是说a、b是同一个东西，因为传入的是引用。这样释放空间时可能会遇到问题。
    - 赋值运算函数
    - 地址运算函数


### 类继承

从一个类派生出另一个类时，原始类成为基类，继承类称为派生类。

- 派生一个类：`class Son: public Base`，将Son类声明为从Base类派生而来
    - 冒号指出Son的基类是Base
    - pulic指出Base是一个公有基类，称为公有派生。
        - 基类的公有成员将成为派生类的公有成员，基类的私有部分也成为派生类的一部分，但只能通过基类的公有和保护方法访问
        - 派生类不能直接访问基类私有成员，必须通过基类的方法进行访问
    - private指出Base是一个私有基类
        - 基类的成员在派生类中中都(公有、保护私有成员)为私有
    - 保护继承
        - 基类的公有、保护私有成员在派生类中中都为保护成员


- 创建派生对象时，程序先创建基类对象。使用下列语法指定创建基类的构造函数。
    - `Son::Son(int age, string name, int Sex): Base(int age){}` 
    - 其中`:Base(int age)`是成员初始化列表。参数从派生类构造函数传入基类构造函数
    - 也可对派生类的成员使用成员初始化列表语法：
        - `Son::Son(int age, string name, int Sex): Base(int age), age(age){}` 
        - 相当于`Son::Son(int age, string name, int Sex): Base(int age){ this.age = age;}`

- 派生类和基类之间的特殊关系
    - 派生类对象可以使用基类的公有方法
    - 基类指针可以不进行显示类型转换的情况下指向派生类对象
        - 基类范围更广，派生类可以是基类。但是基类不能是派生类，因为派生类更具体
    - 基类引用可以不进行显示类型转换的情况下引用派生类对象
    - 虽然基类指针或引用只能调用基类方法

- 多态：方法的行为取决于调用该方法的对象
    - 两种机制实现多态
        - 在派生类中重新定义
        - 使用虚方法
    - 其他容易理解，这里特别介绍虚方法，它将决定指针或引用调用那种方法
        - 如果没有使用虚方法关键字virtual，程序将根据引用类型或指针类型选择方法
        - 如果使用虚方法关键字virtual，程序将根据引用或指针指向的对象的类型选择方法
            - 即"虚方法将看到指针或引用的本质"

- 将源代码中的函数调用解释为执行特定的函数代码块被称为函数名联编(binding)
    - 在编译过程中进行联编被称为静态联编或早联编
    - 在程序运行时进行联编称为动态联编或晚联编(如多态)

- 访问控制：protected
    - protected与private类似，区别之在基类和派生类之间才会表现出来
    - 对外界来说protected成员的行为与private相似;但对于派生类来说protected成员的行为与公有成员相似
        
- **抽象基类(abstract base class, ABC)**
    - 将类的共性抽象出来，放到一个ABC类中，然后再用ABC类派生有这些共性的类
        - 这么做便可以使用基类指针(配合虚函数)数组同时管理多个类的对象了
    - C++通过使用纯虚函数提供未实现的函数。纯虚函数声明的结尾处为=0，原型中的=0是虚函数成为纯虚函数：`virtual void func(int arg) = 0;`
    - 当类声明中包含纯虚函数时，不能创建该类的对象，只能作为基类。成为真正的ABC至少包含一个纯虚函数
    - C++允许在基类中定义(只是不能创建对象)


## C++中代码重用

- 包含对象成员的类
    - has-a
    - 有一些类为表示类中的组建提供了方便，用这些类作为成员以就不需要重复的定义
        - 如`valarray<T>`类，类似`vector<T>`，但valarray提供的算数支持更多，如max、sum。如果类的成员需要算数数组就可用valarray，而不用重新定义数组和sum、max等类方法，因为valarray对象成员包含了这些方法

- 使用using重新定义访问权限
    - 使用保护派生或私有派生时，要让基类的方法在派生类外面可用的一种方法是将函数调用包装在另一个函数调用中，即使用using来指出派生类可以使用特定的基类成员

- 多重继承`class Son: public Dad, private Mom{...};`
    - **虚基类**：使得从多个类(它们的基类相同)派生出的对象只继承一个基类对象，而不是重复的两个
        - `class Singer: virtual public Worker{...};`
        - `class Waiter: public virtual Worker{...};`
        - `class SingerWaiter: public Singer, public Waiter{...};`，SingerWaiter对象将只包含worker对象的副本，而不是引入各自(Singer、Waiter)的Worker对象副本

- 类也有模板类、同样的类成员、友元也是，定义方法同模板函数
    - 有模板类就会有隐式实例化、显式实例化和显式具体化，同模板函数


## Map

### unordered\_map

``` c
unordered_map<Type, Type> Hash;
//索引、添加、修改
Hash[key];
Hash[1] = 2; //添加
Hash[2] = 3; //修改
//计数
Hash.count(key);
//删除
Hash.erase(key);
```



