---
title: rust小炒
author: 66RING
date: 2000-01-01
tags: 
- rust
mathjax: true
---

# Overview

rust中的一些技巧



# tips

## 引用与借用的区别

创建引用的行为称为借用(borrowing)

## 裸指针和引用的相互转换

引用可以隐式或显式地转换为裸指针

- 显式: `let a: *const i32 = &b as *const i32;`
- 隐式: `let a: *cosnt i32 = &b;`

```rust
// Explicit cast:
let i: u32 = 1;
let p_imm: *const u32 = &i as *const u32;

// Implicit coercion:
let mut m: u32 = 2;
let p_mut: *mut u32 = &mut m;
```

**裸指针转引用**: 

1. 指针解引用获取对象: `*p`
2. 获取对象的引用: `&*p`, `&mut *p`
3. 裸指针解引用是unsafe操作

```rust
unsafe {
    let ref_imm: &u32 = &*p_imm;
    let ref_mut: &mut u32 = &mut *p_mut;
}
```


## rust中打印变量地址

先将变量的引用`&val`转换为相应的裸指针`*const T`或`*mut T`，然后再转换成`usize`

```
// 打印裸指针
let mut a: i32 = 5;
println!("{:#x}", &a as *const i32 as usize);

// 通过裸指针修改变量
let p = &mut a as *mut i32;
unsafe {*p = 100};
println!("{}", a);
```


# lang

## 声明式宏

**`macro_rules`**未来将会被废弃

- 使用`#[macro_export]`注释将宏导出
- 使用`macro_rules! NAME {}`进行宏定义

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

- `$x:expr`表示匹配任何表达式，并命名为`x`，使用`$x`访问
- `$()`表示里面的内容会匹配多次，`$(),*`表示有潜在逗号分隔，会匹配多次
- `()=>{}`，不用`$`就只匹配一次
- `{}`中的内容是要展开的表达式
	* 使用`$()*`表示根据匹配次数生成对应代码

