---
title: This和箭头函数
date: 2020-8-20
tags: javascript, typescript
---


## This

**This用于访问当前方法所属的对象**，取决于调用的对象如：

``` javascript
let obj = {
    a: 12,
    fn(){
        console.log(this)
    }
}
// 或者形如
let obj = {
    a: 12,
}
obj.fn = function(){
    console.log(this)
}


obj.fn()  // 将打印obj这个对象
```

上面的例子中，`fn()`输入obj，所以this就是obj


### 调用工具函数来改变this

``` javascript
show(){
    console.log(this)
}
```

- 当直接调用`show()`时，严格模式下this是undefine
- 使用`call`强制改变`this`
    * `show.call(obj)`，那么this就等于obj
    * 与call类似的还有`bind`，`apply`等
        + `f=show.bind(obj)`指定this为obj并返回一个函数


## 箭头函数

普通函数中，this取决于调用;在箭头函数中，this取决于定义。定义箭头函数时，箭头函数中的this取决于当前环境的this，如：

``` javascript
console.log(this)  // {}
const fn=()=>{
    console.log(this)
}
fn()  // {}
// 甚至call也改变不了
fn.call(12)  // {}
```


### 应用举例

如在react中想要点击按钮给数字加1

``` javascritp
// WRONG
<button onClick={function(){
    this.setState({a: this.state.a+1})
}}>按钮</button>
// WRONG 此时传入普通函数相当于直接调用，那么this就是undefine

// CORRECT
<button onClick={()=>{
    this.setState({a: this.state.a+1})
}}>按钮</button>
// CORRECT 此时传入箭头函数，this取决于当前环境，即React.Component这个class中
```


