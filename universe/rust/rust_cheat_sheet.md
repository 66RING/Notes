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


## 错误处理

rust引入了很多现代的抽象来消除Undefine Behavior，如`Result`, `Option`。而这些抽象可能需要很多重复的繁琐的"分支处理"，如：

```rust
match result {
	Ok(o) => o,
	Err(e) => panic("{:?}", e),
}
```

于是Rust由引入了一些方便程序员的函数。上面的代码就得到了简化

- `unwrap()`，对于一个`Result`或`Option`或实现了相关trait的，返回内容物，否则panic
- `expect("")`，类似unwrap，但是可以指定panic的作为信息

对于不让用户自己处理的错误可以"抛出"，即作为函数返回，而不是立刻处理，类似java的throw

```rust
let f = File::open("path");
match f {
	Ok(o) => o,
	Err(e) => {
		return Err(e);
	}
};
```

rust中的简化是使用`?`操作符，如`let f = File::open("path")?`将错误返回。需要注意的是**使用`?`操作符的函数的返回类型必须实现了响应trait**。

- `?`如果"错误"返回出去，否则拿到内容物
- 一个函数可能返回多种错误类型，这样它的返回值可以使用`Box<dyn Error>`，即实现了该trait的对象，或者能实现`from`转换掉

因为`?`的存在，必定会返回一个result，所以**原本无返回的函数还要返回`OK(())`**


## RefCell

> 不可变引用一个改变数据之：内部可变性

"data race = undefine behavior = rust 借用规则"

rust 借用规则(读写锁): 可以存在一个可变引用(借用)或多个不可变引用。rust通过这点来消除undefine behavior

`RefCell<T>`内部通过unsafe代码来维护可变引用和不可变引用的计数，从而绕过编译器借用规则检查，**在运行时执行借用规则检测**，一旦出错将会panic。

- `RefCell<T>::borrow()`返回一个不可变引用(借用)
	* `Ref<T>`计数加1
	* `Ref<T>`离开作用域，计数减1
- `RefCell<T>::borrow_mut()`返回一个可变引用(借用)
	* `RefMut<T>`计数加1
	* `RefMut<T>`离开作用域，计数减1

运行时违反借用规则将panic


