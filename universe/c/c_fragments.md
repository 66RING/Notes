---
title: c/cpp语言碎片知识
date: 2019-12-04
tags: c/c++
mathjax: true
---

# c

## 读写

### 读取字符串

- 读取单个字符
    * `getchar()`是`scanf("%c", c)`的简化版本，除了更简介无其他优势
    * `getche()`，没有缓冲区，输入一个字符后立即读取，不等待用户按回车
    * `getch()`，没有缓冲区，输入一个字符后立即读取，不等待用户按回车，区别于`getche`它**没有回显** 
- 读取字符串
    * `scanf("%s", str)`遇到空格后停止接收后面的字符串
    * `gets()`有缓冲区，可以读取空格，直到回车才会结束
    * `scanf("%[^\n]",str)`，可以读取带空格的字符串，直到回车才结束


### 文件读写

#### 打开/关闭文件

打开或新建一个文件

```c
FILE *fopen( const char * filename, const char * mode );
```

- mode:
    * r, **只读**方式打开，文件必须存在
    * w, **只写**方式打开，文件存在则清空文件内容从头写，否则新建并打开
    * a, **只写追加**
    * r+, **可读写**方式打开，不同w会清空文件内容，文件必须存在
    * w+, **可读写**方式打开，其他同w
    * a+, **可读写**方式打开，其他同a
    * 如果打开二进制文件需要加个`b`，如`rb`


关闭文件

```c
int fclose(FILE *fp);
```

如果关闭发生错误返回EOF


#### 写入文件

写入一个字符，成功返回写入的字符，错误返回EOF

```c
int fputc( int c, FILE *fp );
```

写入字符串，成功返回非负，错误返回EOF

```c
int fputs( const char *s, FILE *fp );
// or 格式化写入
int fprintf(FILE *fp,const char *format, ...)
```


#### 文件读取

读取一个字符，成功返回该字符，错误返回EOF

```c
int fgetc( FILE * fp );
```

从流中读取n-1个字符并拷贝到buff，并追加一个null终止字符串。如果读到`\n`或EOF则会提前返回

```c
char *fgets( char *buf, int n, FILE *fp );
// or
int fscanf(FILE *fp, const char *format, ...)
// 函数来从文件中读取字符串，但是在遇到第一个空格和换行符时，它会停止读取。
```


#### 重定向文件流

TODO

```c
#include<stdio.h>
FILE *freopen(char *filename, char *mode, FILE *stream);
```

- filename，欲打开的文件路径
- mode
- stream，已经打开的文件指针，如stdin，stdout等

tips:

```c
// 把in.txt内容作为标准出入
freopen("in.txt","r",stdin);
// 把标准输出重定向到out.txt
freopen("out.txt","w",stdout);
```


## 指针

### anywhere

常数是是不能做指针的，所有要用"常数指针"，需要先将常数强制转换成指针类型，如下将0转换成了一个函数指针。

```
(void(*)())0
```

之后可以像调用函数指针一样调用：

```
(*(void(*)())0)()
```


### 函数指针

函数指针声明如下

```c
返回值类型(*指针变量名)(参数列表)
int(*func_ptr)(int, int) = & func; // &可以省略
```

上面声明了一个指向 *返回值为int，有两个int类型参数的函数* 的指针`func_ptr`，接下来就可以通过`func_ptr(int, int)`调用了


#### 回调函数

函数指针变量作为某个函数的参数，即别人的函数执行时调用你实现的函数


### void\*

void指针可以指向任意类型的数据，**赋值给void指针无需强制类型转换** 。但是void指针赋值给其他类型就需要强制类型转换了。一个常见的例子就是`malloc`函数，它就是返回一个`void*`，这给使用带来了很大的灵活性。

利用void指针的性质，可以用在函数中接收任意参数。常见的如内存操作`void * memcpy(void *dest, const void *src, size_t len)`。显然内存才不管什么类型，只管操作一片区域。

- ANSI C标准不允许对void指针进行如`++`等的操作，因为无类型就不知道一次操作几个字节
- GNU标准运行对void指针进行运算操作，认为`void*`和`char*`是一样的


## 变量


### static

static有如下功能

- 1. 隐藏变量
    * 对于未加static的全局变量和函数具有全局可见性，可在另一个文件中`extern`关键字声明使用
- 2. 保持变量内容的持久
    * 存储在静态数据区的变量会在程序运行时就初始化，且也是唯一一次初始化。全局变量和static变量就是静态数据区的变量
    * 不同于全局变量，static可以控制变量的可见范围，也就是隐藏局部变量，但局部变量从储存是全局的
- 3. 初始化为0
    * 静态数据区中所有的字节默认都是0x00


### extern关键字

- 声明extern关键字的全局变量和函数可以被跨文件访问
- `extern "C"`，具备extern特性的同时，按照C语言的方式编译连接(如在C++中使用)


### 枚举enum

用符号名称代表常数，提高程序可读性。枚举常量是int类型的，因此任何int类型使用的地方都可以用它。定义枚举类型方法如下

```c
enum 枚举名 {枚举元素1, 枚举元素2,...};
```

**需要注意的是** 第一个枚举元素默认为0，后续元素在前一个元素上加1。

可以在声明时指定枚举元素的值，没指定值的元素会在前一个元素的值上加1

```c
enum H {A, B=3, C, D=7, E};
// A=0, B=3, C=4, D=7, E=8
```


#### 定义枚举变量

枚举变量是值可以用枚举类型中定义的名称赋值(当然也可以用任意int或unsigned int赋值，但这么做就失去了可读性)，如`enum day = MON`。定义枚举变量有三种方式

```c
// 先定义枚举类型再定义枚举变量
enum DAY
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
}; 
enum DAY day;

// 先定义枚举类型同时定义枚举变量
enum DAY
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
} day; 

// 省略枚举类型名，直接定义枚举变量
enum 
{
    MON=1, TUE, WED, THU, FRI, SAT, SUN
} day; 
```


## 编译

### 宏

简单的讲，宏就是把一个缩略语(宏，macro)指定成任何一段文本。预处理时会用定义和文本把这个缩略语替换掉。

- 没有参数的宏
    * `#define 宏名称 文本`
- 带参数的宏
    * `#define 宏名称(参数列表) 文本`
    * **注意** 宏参数的替换也是文本的替换，因此不要吝啬括号
        ```c
        #define T(x, y) x*y
        T(1+1, 2) // 结果是1+1*2 = 3
        #define T(x, y) (x)*(y) // 这样才能得到4
        ```
- 可选参数的宏
    * `#define 宏名称(参数列表,...) 文本`
        + 省略号必须放在参数列表的后面，表示可选参数
    * 可选参数会用**连同分隔他们的逗号**打包在`__VA_ARGS__`中
- 字符串化运算符
    * 形参中使用`#`可以将宏中的实参转换用双引号`"`包裹成字符串
        + 如果实参有双引号`"`，则每个双引号前面会加反斜杠`\`
        + 同理反斜杠以后自动转意
        + **PS** 编译器会合并紧邻的字符串字面量(`"ab""c" -> "abc"`)
- **记号粘贴运算符**
    * `##`，会把左、右操作数组合在一起作为一个记号
        + `#define A(x) a_##x -> A(1) -> a_1`
        + 出现在`##`两边的空白符会连同`##`一起删除
        + 如果得到的结果是宏则预处理器继续进行宏替换，即 **记号粘贴完成后才进行宏展开**


### 条件编译

不同平台提供的api不同，根据平台编译代码是必须的

`#if`, `#elif`, `#else`, `#endif`都是预处理命令，这些操作都是在预处理阶段完成的，多余的代码以及宏都不会参与编译，保证正确性还缩小了体积。

`#ifdef`，意思是如果宏被定义过，`#ifndef`则是它的非

需要注意的是`#if`后面跟的是"整型常量表达式"，而`#ifdef`和`#ifndef`后面只能跟宏名



## goto

跳转然后继续顺序执行。

- 向后跳转

```c
goto label;
..       |
..       |
label:  <
    statement;
```

- 向前跳转

```c
label:         <-|
    statement    |
..               |
..               |
goto label;     -
    statement;
```


## 系统调用

### mmap

```c
void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize);
```

- 主要用途
    * 将普通文件映射到内存，这样频繁读写时就可直接在内存读写，提高性能
    * 为关联的进程提供共享内存空间
    * 为无关联的进程提供共享内存空间
		+ `fd = shm_open("/path")`
		+ `mmap(fd)`
- 参数说明
    * start：指向内存起始地址，为NULL则让系统自动选定，映射成功返回该地址
    * length：文件中多大的部分映射到内存
    * prot：映射区域的保护方式，可以是以下几种方式的组合：
        + `PROT_EXEC`，映射区域可被执行
        + `PROT_READ`，映射区域可被读取
        + `PROT_WRITE`，射区域可被写入
        + `PROT_NONE`，映射区域不能存取
    * flags：会影响映射区域的各种特性
        + 在调用mmap()时必须要指定`MAP_SHARED` 或`MAP_PRIVATE`
        + `MAP_FIXED`，如果参数 start 所指的地址无法成功建立映射时，则放弃映射，不对地址做修正。通常不鼓励用此旗标。
        + `MAP_SHARED`，对应射区域的写入数据会复制回文件内，而且允许其他映射该文件的进程共享。
        + `MAP_PRIVATE`，对应射区域的写入操作会产生一个映射文件的复制，即私人的"写入时复制"对此区域作的任何修改都不会写回原来的文件内容。
        + `MAP_ANONYMOUS`，建立匿名映射，此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享。
        + `MAP_DENYWRITE`，只允许对应射区域的写入操作，其他对文件直接写入的操作将会被拒绝。
        + `MAP_LOCKED`，将映射区域锁定住，这表示该区域不会被置换(swap)。
    * fd：`open()`返回的文件描述符，代表欲映射到内存的文件
    * offset：文件映射的偏移量，0表示从最前方开始，offset必须是分页大小的整数倍
    * 返回值：若映射成功则返回映射区内存的起始地址，否则返回`MAP_FAILED`(-1)
- 解除内存映射
    * `int munmap(void *start, size_t length)`
    * 取消从`start`起`length`大小的内存映射
    * 关闭文件描述符不会自动解除映射
- 错误代码
    * `EBADF`，参数fd 不是有效的文件描述词
    * `EACCES`，存取权限有误。如果是`MAP_PRIVATE`情况下文件必须可读，使用`MAP_SHARED`则要有`PROT_WRITE`以及该文件要能写入。
    * `EINVAL`，参数start、length 或offset有一个不合法。
    * `EAGAIN`，文件被锁住，或是有太多内存被锁住。
    * `ENOMEM`，内存不足。


### sleep

```c
unsigned int sleep(unsigned int seconds);
```

- linux下`sleep()`函数在`#include<unistd.h>`中
- windows下`sleep()`函数在`#include<windows.h>`中
- 跨平台
    ```c
    #ifdef _WIN32
    #include <Windows.h>
    #else
    #include <unistd.h>
    #endif
    ```

需要**注意**的是`printf()`是一个**行缓冲函数**，先写到缓冲区，满足一定条件后才，才刷新。刷新缓冲区的条件如下：

- 缓冲区填满
- 写入字符有`\n`，`\r`
- 调用`fflush`手动刷新时
- 调用`scanf`需要从缓冲区读取数据时


### exec

#### execv

```c
//#include <unistd.h>
int execv(const char *filename, char *const argv[]);
```

以新进程(filename)替换当前进程，pid不变。如果正常执行，`execv`永远不会返回

- filename：二进制可执行程序
- arg：传入被执行程序的参数序列

execve同理，可以携带环境变量参数序列

```c
//#include <unistd.h>
int execve(const char *filename, char *const argv[], char *const envp[]);
```

- filename：二进制可执行程序
- arg：传入被执行程序的参数序列
- envp：环境变量序列

```c
#include<stdio.h>
#include<unistd.h>

int main(int arg, char **args)
{
    char *argv[]={"python","./test.py",NULL};
    char *envp[]={0,NULL};
    execve("/usr/bin/env",argv,envp);
}
```


## 结构体初始化

机构体初始化有4种方法，观察如下结构体：

```c
struct Person{
    int age;
    char* name;
    int weight;
}
```

法一：定义时顺序赋值

```c
struct Person p = {18, "ring", 110};
```

法二：定义后赋值

```c
struct Person p;
p.age = 18;
p.weight = 110;
p.name = "ring"
```

法三：定义时乱序赋值(C风格)，**注意使用的是逗号隔开** 

```c
struct Person p = {
    .age = 18,
    .weight = 110,
    .name = "ring"
}
```

法四：定义时乱序赋值(C++风格)

```c
struct Person p = {
    age : 18,
    weight : 110,
    name : "ring",
}
```


## union共用体

共用体定义方法类似结构体：

```c
union union_name {
	type1 member1;
	type1 member2;
	type2 member3;
} value;
```

其中共用体名`union_name`和变量名`value`都是可省的。省略共用体名`union_name`则声明变量只能在定义共用体时，即之后都不用这个共用体声明变量了。省略变量名`value`，则相当于定义了一种类型，之后可以类似结构体一样通过`union union_name value`声明变量。

共用体是一种多个变量共用同一块内存空间的类型，**一次只能有一个成员有效** ，而共用体的大小就等于最大成员的大小。

应用：当几个个结构中大部份都相同而只有少数不同时，可以不重新写一个差不多一样的结构体，而是在结构体中使用共用体。如：

```c
union teacher_or_student{
	char name[20];
	union {
		int score;
		char course[20];
	};
	// 如果是学生则使用成绩，score成员
	// 如果是教师则使用课程，course成员
}
```

## typedef的两种用法

- 1. 定义类型别名`typedef TYPE alias_name`
- 2. 定义函数指针别名，如`typedef int(*func)()`，则func类型就为一个返回int，无参数的函数指针类型

# cpp

## cpp类初始化

### 初始化列表

写法:

```cpp
class A{
private:
    int Age;
    string Name;
public:
    A(int age, string name):
        Age(age),
        Name(name)
    {}
};
```

推荐类初始化使用初始化列表，如果构造函数写出这样：

```cpp
A(int age, string name){
    Age = age;
    Name = name;
}
```

这种做法不是最佳的，是赋值操作（伪初始化）。因为初始化发生在构造函数本体之前，default构造函数(即调用类型的拷贝函数)为成员设初值，然后再立即赋值，这样显然default构造函数的操作就浪费了。因此使用初始化列表只调用一次设值，比较高效。





