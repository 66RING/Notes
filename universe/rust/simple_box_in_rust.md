---
title: rust中指针指针box的简单实现
author: 66RING
date: 2022-11-10
tags: 
- rust
mathjax: true
---

# rust中指针指针box的简单实现

## Abstract

box是rust中最基本的一种智能指针, 所谓智能能自动解引用自动释放。这三个基本功能分别是通过实现`Deref`, `DerefMut`, `Drop`三个trait赋能的。

`Deref`和`DerefMut`实现了智能指针的自动解引用。简单的说是自动解引用直到类型匹配(如果一路都满足Deref trait)，即如果需要的类型是`&T`而得到的类型是`Box<T>`, 那么`Box<T>`就会调用`deref()`自动转换出`&T`。比如一个函数`f(t: &T)`需要`&T`, 我们传入的是一个`&box<T>`: `f(&b)`, 那么Box就会自动解引用出内部数据T。

自动释放则是利用rust生命周期的特性, 在生命周期结束时自动调用`Drop`trait的`drop()`释放。


## 一个简单的实现

我们的简单box(就称之为`SBox`吧)需要实现四个功能

- `new()`, 从堆申请内存并保存到结构体内部
- `deref()`/`deref_mut()`, 自动对`SBox`解引用
- `drop()`, 生命周期结束后自动将内存还给堆
- `into_raw()`, 退去智能指针的包裹, 拿到原始是裸指针

使用一个大数组模拟堆, 数组的每个元素都是一个Page, 一个Page有4096个Byte。逻辑比较简单, 申请内存就是把数组中一个元素的地址返回, 用于模拟物理空间的申请。我们这里为了方便检测, 值运行顺序释放, 通过`assert_eq!()`判断释放的就是刚刚分配出去的。


```rust
#[repr(align(4096), C)]
struct Page([u8; 4096]);

impl Page {
    pub const fn new() -> Self {
        Self([0; 4096])
    }
}

struct Heap {
    pub data: Vec<Page>,
    pub len: usize,
}

impl Heap {
    pub const fn new() -> Self {
        Self {
            data: vec![],
            len: 100,
        }
    }

    pub fn init(&mut self) {
        for _ in 0..100 {
            self.data.push(Page::new());
        }
        self.len = 100;
    }

    pub fn kalloc(&mut self) -> usize {
        if self.len <= 0 {
            panic!("out of memory");
        } else {
            self.len -= 1;
            &self.data[self.len] as *const _ as usize
        }
    }

    pub fn kfree(&mut self, pa: usize) {
        assert_eq!(pa, &self.data[self.len] as *const _ as usize);
        self.len += 1;
    }
}

static mut HEAP: Heap = Heap::new();
```

官方Box使用内部指针使用`Unique<T>`封装, 我们使用stable版的`NonNull<T>`。新建只能指针的时候从堆中申请一块物理地址pa, 将该物理地址强制转换成另一中类型的指针`*mut T`保存在`NonNull<T>`中。方便后续调用`NonNull<T>`的方法。相当于`NonNull<T>`帮我们维护了内部的裸指针, Box帮我们做一些智能的操作。

自动解引用方面, 调用`NonNull<T>`的方法返回引用, 它会对内部裸指针解引用拿到对象后使用`&`符返回引用。形如`&mut *ptr`, ptr是某个指针。基于这点你也可以实现一种不用NonNull的版本。

获取裸指针方面, 使用`NonNull<T>`的方法返回内部裸指针

自动释放方面, 拿到裸指针后调用`kfree()`归还给堆。并且通过了`assert_eq!()`, 说明是刚刚分配的内存。


```rust
struct SBox<T>(NonNull<T>);

impl<T> SBox<T> {
    pub fn new() -> Self {
        unsafe {
            let pa = HEAP.kalloc();
            Self(NonNull::new(pa as *mut T).unwrap())
        }
    }

    pub fn into_raw(&self) -> *mut T {
        unsafe { self.0.as_ptr() }
    }
}

impl<T> Deref for SBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe { self.0.as_ref() }
    }
}
impl<T> DerefMut for SBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { self.0.as_mut() }
    }
}

impl<T> Drop for SBox<T> {
    fn drop(&mut self) {
        unsafe { HEAP.kfree(self.0.as_ptr() as usize) }
    }
}
```

测试程序如下: 用我们的简易Box创建了一个只能指针a, 打印对空间大小发现确实减一到了99。然后后续函数接收的参数都是内部类型`&i32`, 而我们传入的啊`&a`, 类型为`&Box<i32>`，这时只能指针就会自动解引用至类型匹配。

最后花括号指定的生命周期结束, 堆大小恢复100

```rust
fn main() {
    unsafe {
        HEAP.init();

        {
            let mut a = SBox::<i32>::new();
            println!("heap len {}", HEAP.len);
            *a = 5;
            println!("{}", *a);
            fn t_ref(x: &i32) {
                println!("ref {}", x);
            }
            fn t_mut(x: &mut i32) {
                println!("mut {}", x);
            }
            t_ref(&a);
            t_mut(&mut a);

            unsafe {
                println!(
                    "{:#x} {:#x}",
                    a.into_raw() as usize,
                    &HEAP.data[HEAP.len] as *const _ as usize
                )
            };
        }
        println!("heap len {}", HEAP.len);
    };
}
// $ cargo run
// heap len 99
// 5
// ref 5
// mut 5
// 0x7f7eb5b1a000 0x7f7eb5b1a000
// heap len 100
```





