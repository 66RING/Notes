---
title: 重载是如何实现的
date: 2020-11-15
tags: tech, c, gcc
---

> overload可直译为重载，它是指我们可以定义一些名称相同的方法，通过定义不同的输入参数来区分这些方法，然后再调用时就会根据不同的参数样式，来选择合适的方法执行。
>
> --百度百科


## 原理

重载本质上就是编译器根据原函数名和参数类型对原函数名进行改编，以区分接受不同参数的同名函数。

看下面一段cpp代码：

```cpp
// a.cpp
void func(int a){
}

void func(int a, int b){
}

int main(){
    func(1);
    func(1, 2);
}
```

规定是不能定义两个同名函数的，但是如果是不同参数的同名函数就可以，这就是重载的作用。

我们通过`objdump`查看反汇编查看源码来理解其中的原理。

```shell
gcc -c a.cpp -o a.o
objump -d a.o
```

- `gcc -c a.cpp -o a.o`
    * 这条命令用来汇编生成的目标文件`.o`结尾
- `objdump -d a.o`
    * `-d`反汇编我们刚才生成目标文件

反汇编后将会看到汇编代码，我们抓取我们的重点：接受不同参数的同名函数

```shell
0000000000000000 <_Z4funci>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   89 7d fc                mov    %edi,-0x4(%rbp)
   7:   90                      nop
   8:   5d                      pop    %rbp
   9:   c3                      retq

000000000000000a <_Z4funcii>:
   a:   55                      push   %rbp
   b:   48 89 e5                mov    %rsp,%rbp
   e:   89 7d fc                mov    %edi,-0x4(%rbp)
  11:   89 75 f8                mov    %esi,-0x8(%rbp)
  14:   90                      nop
  15:   5d                      pop    %rbp
  16:   c3                      retq
```

上面看到我们定义的两个同名函数`<_Z4funci>`和`<_Z4funcii>`，仔细一看好似有规律可寻。

- `_Z`后跟的数字表示函数名的长度和函数名，这里是4对应后面`func`的长度
- 在函数名的后面跟的就是参数的类型，这里我们分别是一个整形，和两个整形的函数所以就是`i`和`ii`

所以编译器通过这么一个简单的对函数名的改编(name mangling)就可以做到区分接受不同参数的同名函数的功能。

**tips:** 使用`c++filt`可以查看原函数名。

看这么一个函数`void func(s string){}`，反汇编后的函数名为`<_Z4funcNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE>`。

使用`c++filt`查看其原函数名：`c++filt _Z4funcNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE`。

```shell
$ c++filt _Z4funcNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE 
func(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >) 
```



