---
title: rust程序冲突tips
author: 66RING
date: 2022-11-11
tags: 
- rust
mathjax: true
---

# rust程序冲突tips

对[这篇文章](http://smallcultfollowing.com/babysteps/blog/2018/11/01/after-nll-interprocedural-conflicts/)的二次加工和翻译。


## 问题

我们想要将程序的部分功能抽象成一个"管理者"。 于是我们会构造一个结构体, 它内部有一系列"被管理元素"和一些状态元素。如:

```rust
use std::sync::mpsc::Sender;

struct MyStruct {
  widgets: Vec<MyWidget>,
  counter: usize,
  listener: Sender<()>,
}

struct MyWidget { .. }

impl MyStruct {
  fn signal_event(&mut self) {
    self.counter += 1;
    self.listener.send(()).unwrap();
  }
}
```

我们管理若干Widget(如gui组件), 并在有事件发生是`counter += 1`, 并通知`litsener`, 即`signal_event`方法。我们的方法是遍历所有widget检查是否有事件发生`check()`，然后做相应操作。

```rust
impl MyStruct {
  fn check_widgets(&mut self) {
    for widget in &self.widgets {
      if widget.check() {
        self.signal_event();
      }
    }
  }
}
```

会发现编译失败。那是因为我们访问内部"被管理者"时需要持有self的所有权。然后如果我们还要`signal_event`的话又需要`&mut self`。这样就违反了rust的借用规则。

```rust
error[E0502]: cannot borrow `*self` as mutable because it is also borrowed as immutable
  --> src/main.rs:30:9
   |
28 |     for widget in &self.widgets {
   |                   -------------
   |                   |
   |                   immutable borrow occurs here
   |                   immutable borrow later used here
29 |       if widget.check() {
30 |         self.signal_event();
   |         ^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

以上就是这么一个大结构体带来的问题。


## 解法1：内联

把两个冲突的方法合并。这样以来我们就有多分相同的逻辑: 亮出`signal_event()`相同的逻辑, 复杂项目中就容易导致不同步。 称之为"DRY"-failure

```rust
impl MyStruct {
  fn check_widgets(&mut self) {
    for widget in &self.widgets {
      if widget.check() {
        // Inline `self.signal_event()`:
        self.counter += 1;
        self.listener.send(()).unwrap(); 
      }
    }
  }
}
```

## 解法2: 重构结构体

我们可以拆分成这样。这样我们就不会持有两个MyStruct了, 而是一个MyStruct一个EventSignal。但是, 我们我们额外需要添加一个处理`counter`和`widgets`的逻辑呢? 显然没有办法。

不过将"管理者"拆分从"被管理者"和"属性"基本可以解决大多数问题了。

```rust
struct EventSignal {
  counter: usize,
  listener: Sender<()>,
}

impl EventSignal {
  fn signal_event(&mut self) {
    self.counter += 1;
    self.listener.send(()).unwrap();
  }
}

struct MyStruct {
  widgets: Vec<MyWidget>,
  signal: EventSignal,
}
```

## 解法3: 面向过程

`signal_event`不再是对象的方法, 而是一个独立的函数。使用时逐个传入成员。很灵活, 但是有点"开倒车", 而且需要手动传很多参数。

```rust
fn signal_event(counter: &mut usize, listener: &Sender<()>) {
  *counter += 1;
  listener.send(()).unwrap();
}
```

## 结论

没有通用的方法，问题在于自己清楚需要用到哪些成员，要怎么用。上述一些方法可以根据自己实际情况使用。


