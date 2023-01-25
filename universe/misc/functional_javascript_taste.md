---
title: 通过javascript体会函数式编程
author: 66RING
date: 2022-12-18
tags: 
- misc
- functional programming
mathjax: true
---

# Abstract

> 虽然像递归, 但是你要用functional的眼观看待

- 对js做7点限制, 从而体会函数式
    1. no loops
    2. no if
    3. single return
    4. no side-effects to outside world
    5. no assignments in functinos
    6. no arrays
    7. only 0 or 1 argument

- 多参数
- 链表
    * first, second, ...
    * head, next
- 数组
- list2array
- array2list
- range
- ⭐ map


## 多参数逻辑

> 函数式的上下文在闭包中

加法为例

```javascript
let add = (a) => (b) => a + b
```

## pair

> 有了pair的概念后就可以递归了

```javascript
let pair = (first) => (second) => {
    return {
        first: first,
        second: second,
    }
}
```

取成员

```javascript
let fst = (pair) => pair.first
let snd = (pair) => pair.second
fst(pair(1)(2))
snd(pair(1)(2))
```


## 链表

> 链表是如何递归的

利用pair做一个手动链表

```javascript
let list = pair(1)(pair(2)(pair(3)(null)))
```


## list2array

有了pair的概念, 迁移到链表上就是: `data`, `next`。所以可以创建两个辅助函数`data`和`next`

```javascript
let data = (p) => p.first
let next = (p) => p.second

let list2array = (node) => {
    let result = []
    while (node !== null) {
        result.push(data(node))
        node = next(node)
    }
    return result
}
```


## array2list

> 创node, 接node

不够我们这里要从尾到头创建: 递归的尽头是next

```javascript
let array2list = (a) => {
    let xs = Array.from(a).reverse()
    let node = null
    for (let i=0; i<xs.length; i++) {
        node = pair(xs[i])(node)
    }
    return node;
}
```

## range

给定一个范围自动创建一range的链表

```javascript
let range = (low) => (high) => 
    low > high ? 
    null : pair(low)(range(low+1)(high))

list2array(range(1)(100))
```


## map

> map(f)(list), 将f作用于list中的所有元素然后返回新list

之所以抽象成list是因为我们想尽可能用函数式的方式遍历

```javascript
let map = (f) => (list) => {
    return list === null 
        ? null 
        : pair(f(data(list))) (map(f)(next(list)))
}

list2array(map((x) => x*2)(range(1)(10)))
```

## 总结

- 多参数传递的模式: 
    * 不管三七二十一, 先把函数参数保存到闭包中再说
    * 也就相当于`(arg1, arg2, ...)`变成了`(arg1)(arg2)(...)`

```javascript
let pair = (first) => (second) => {
    return {
        first: first,
        second: second,
    }
}

let fst = (pair) => pair.first
let snd = (pair) => pair.second

let list = pair(1)(pair(2)(pair(3)(null)))

let data = (p) => p.first
let next = (p) => p.second

let list2array = (node) => {
    let result = []
    while (node !== null) {
        result.push(data(node))
        node = next(node)
    }
    return result
}

let array2list = (a) => {
    let xs = Array.from(a).reverse()
    let node = null
    for (let i=0; i<xs.length; i++) {
        node = pair(xs[i])(node)
    }
    return node;
}

let range = (low) => (high) => 
    low > high ? 
    null : pair(low)(range(low+1)(high))

list2array(range(1)(100))

let map = (f) => (list) => {
    return list == null 
        ? null 
        : pair(f(data(list))) (map(f)(next(list)))
}

list2array(map((x) => x*2)(range(1)(10)))
```

