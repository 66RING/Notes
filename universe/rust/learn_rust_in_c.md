---
title: 用C的方式理解rust
author: 66RING
date: 2000-01-01
tags: 
- rust
mathjax: true
---

# Preface

作为一个母语是c程序员，初见rust时很多改变是比较模糊的，c中的指针等在rust中似乎变得更加抽象难以理解了。所以打算用类比的方式，让c程序员更好的理解rust。

```rust
// 基于引用创建裸指针
let mut num = 5;
let r1 = &num as *const i32;
let r2 = &mut num as *const i32;
println!("{:?} {:?}", r1, r2);

// 基于地址创建裸指针
let addr = 0xdeadbeefusize;
let r3 = addr as *const i32;

// 基于智能指针创建裸指针
let a = Box::new(1);
// 同理基于引用创建裸指针
let b: *const i32 = &*a;
let c = &*a as *const i32;
// 使用api创建
let d: *const i32 = Box::into_raw(a);
```

## mut和&mut

以下操作的区别

```
let a = &b;
let a = &mut b;
let mut a = &b;
let mut a = &mut b;
```

- 第一个表示a是指向b的指针, 指针指向的东西不可变，指针指向不可变
- 第二个表示a是指向b的指针, 指针指向的东西可变，指针指向不可变
- 第三个表示a是指向b的指针, 指针指向的东西不变，指针指向可变
- 第二个表示a是指向b的指针, 指针指向的东西可变，指针指向可变
- `let mut a: &mut T`前一个mut表示a指向可变，后一个mut表示a指向的东西可变(可变引用)

```
let mut b = vec!(1, 2, 3);
let mut c = vec!(1, 2, 3);
let a = &b;
// a[1] = 3; 				// 指向的东西(b)不可变, a的指向不可变
// ==========
let a = &mut b;
a[1] = 3;					// 指向的东西(b)可变, a的指向不可变
// a = &mut c;
// ==========
let mut a = &b;
a = &c;						// a的指向可变
// a[1] = 3;				// 指向的东西不可变
// ==========
let mut a = &mut b;
a[1] = 3;
a = &mut c; 				// <====== 必须&mut c，因为没有用let重新变量绑定，a的类型是可变引用，所以只能接收可变引用
```

- `*const T`, `*mut T`裸指针
	* 不保证有效内存，不保证非空，没有自动清理，不移动所有权，不判生命周期
	* 解引用裸指针是`unsafe`操作
	* 显示`*const T`, `*mut T`声明可变性，不能`*T`

```
let x = 5;
let raw = &x as *const i32;
unsafe {
	println!("{}", *raw);
	// *raw = 3;				// *const不可变
	// println!("{}", *raw);
}

let mut y = 10;
let raw_mut = &mut y as *mut i32; // *mut 可变
unsafe {
	println!("{}", *raw_mut);
	*raw_mut = 3;
	println!("{}", *raw_mut);
}
```

## const 和 static

不用`let`变量绑定的方式

```
const a: i32 = 4;
static a: i32 = 4;
static mut a: i32 = 4;
```

- `const`常量会内联到用到它的地方，没有固定地址
- `static`静态变量由固定地址，不内敛
- 访问`static mut`是`unsafe`的，因为全局变量可能被多个线程使用

