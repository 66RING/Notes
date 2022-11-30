---
title: 浅学WASM栈虚拟机
author: 66RING
date: 2021-10-29
tags: 
- wasm
mathjax: true
---

# WASM栈虚拟机

## WASM简介

- wat是比较可读的汇编文件
- wasm的wat编译得到的结果

顾名思义, 它是某种汇编语法: 即人类不友好, 但机器友好。对人类比较友好的格式就称为`WAT`(WebAssembly Text Format)形如:

```wat
(module
    (func $main
        i32.const 3
        unreachable
    )
    (start $main)
)
```

汇编代码还需要编译成二进制才能够执行, wasm的二进制就称为(后缀)`wasm`。我们可以使用本地的`wat2wasm`也可以使用在线[编译](https://webassembly.github.io/wabt/demo/wat2wasm/)

编译好的二进制文件丢入wasm虚拟机执行。


## 基于栈的虚拟机

> JVM也是stack base的

回想用C语言实现一个计算器的算法, 很多都是使用栈的方式实现的。比如a + b + c, 操作数a, b, c分别压栈, 然后对与操作符号+, 我另栈顶的b,c出栈, 然后将`b+c`结果压栈。

stack base的虚拟机的原理就类似上述过程。


### 指令

#### 一元运算

每次从栈中取出一个操作数, 然后将运算结果压栈。如统计二进制1的个数等

- i32.clz: 最左边连续1的个数
- i32.ctz: 最右边连续1的个数
- i32.popcnt: 二进制1的个数(32bit长)
- i32.eqz: 整数为0则压入1, 否则压入0
- 同理i64
- 其他

```wast
(module
    (func $main
        i32.const 2248752  ;; 00000000 00100010 01010000 00110000
        i32.clz 
        unreachable
        i32.const 2248752
        i32.ctz 
        unreachable
        i32.const 2248752
        i32.popcnt 
        unreachable
        i32.const 2248752
        i32.eqz
        unreachable
        i32.const 0
        i32.eqz
        unreachable
    )
    (start $main)
)
```


#### 二元运算

每次从栈中取出两个操作数, 然后将运算结果压栈。如加减乘除与或非等。虽然栈的后进先出, 但是wasm会稍微调整一下顺序, 如果`a/b`栈顶为b, 如果结果是`b/a`就出问题了。

- i32.add
- i32.sub
- i32.mul
- f32.div
- i32.rotl循环左移
- ...

```wat
(module
    (func $main
        i32.const 16
        i32.const 11
        i32.add
        unreachable
        i32.const 16
        i32.const 11
        i32.sub
        unreachable
    )
    (start $main)
)
```

#### 跳转指令与Block

> stack base怎么做跳转呢?

![](https://781946743-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LOgDUpUHY3kAm86Z3rz%2F-LOgDkeS2P_JfyJegaqb%2F-LOgDlOVBA4RyA6jJgmM%2Fblock.png?generation=1539414128058744&alt=media)

一个简单是理解是类比函数调用: 函数调用会开辟自己的调用栈, 结束后清空。所以就是说stack base vm可以在跳转指令时开辟一个空间(block)去执行, block执行完成可以将执行结果压栈, 然后回到上一层block。

```wat
(module
    (func $main
        block $name (result i32)
            i32.const 5
        end
        unreachable
    )
    (start $main)
)
```

- `$`标记的是label名, 后续跳转会用到
- `(result i32)`标记返回值


### 存储空间

> 虽说是stack base, 但是对于要持久存储的数据还是要有一个独立空间的

全局变量声明

```
;; 不可变匿名全局变量
(module
  (global i32 i32.const 5)
)
;; 可变匿名
(module
  (global (mut i32) i32.const 5)
)
;; 命名全局变量
(module
  (global $name (mut i32) i32.const 5)
)
```

局部变量只能在函数内部不区分可变性, 也是自动从0编号, 也可以使用`$`命名

```
(module
  (func (local $aaa i32) (local $bbb f32) (local i32)
    i32.const 5
    drop
  )
)
```

变量自动从0编号, 使用特殊指令获取: `get_global id`, `set_global id`, `get_local id`等


#### 堆(内存)

```
;; 指定初始的(最小, 最大)值
(module
    (memory 1 5)
)

;; 另一种声明方式
(module
    (memory (data "Hello\0A\00"))
)


(module
    (memory (data "\0A\01\00\00\00"))
    (func $main
        i32.const 0
        i32.load offset=1 ;; 加载到栈上的变量中, 即const 0
        unreachable
        i32.const 0
        i32.load
        unreachable
    )
    (start $main)
)
```

可以使用指令增加内存和读写内存

#### 函数

格式如下, 同样是自动编号, 使用`call id`调用

```
(module
    ;; 有名局部变量
    (func $bbb (param $a i32) (param f32) (param $b i32)
        get_local $a
        get_local $b
        i32.add
        get_local 1
        unreachable
    )

    (func $aaa (param i32 f32 i32) 
        get_local 0
        get_local 2
        i32.add
        get_local 1
        unreachable
    )
    (func $main 
        i32.const 5
        f32.const 3.14
        i32.const 8
        call $aaa
        i32.const 3
        unreachable
    )
    (start $main)
)
```

### 略

