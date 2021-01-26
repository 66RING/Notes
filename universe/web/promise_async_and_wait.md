---
title: Promise, async和wait的简单记录
date: 2020-10-03
tags: 
- web
- promise
- javascript
---

## 为什么需要promise

有时候需要向函数中传入回调函数(异步操作结束后执行)，这么一来可能就会有多重函数嵌套的情况。这样的问题很明显：

- 耦合度高
- 解耦要传函数指针
    * 安全考虑还需判断参数类型，可读性差
    * 如果传送匿名参数，可读性更差


## Promise写法

```javascript
function callRing(success) {
    return new Promise((resolve, reject) => {
        console.log("calling Ring ...")
        setTimeout(() => {
            if (success) {
                resolve()
            }else{
                reject()
            }
        }, 3000)
    })
}
callRing(true)
    .then(()=>{}) // promise对象上的then方法。传入resolve函数
    .catch(()=>{}) // promise对象上的catch方法。传入reject函数
    // 由于返回了promise对象，所以多次then调用是可行的
```

- `.then(function)`，传入promise对象的参数resolve和reject
    * 当然，可以传入两个参数，即第二个参数就是reject。用来指定reject回调
- `.catch(function)`
    * 它就是then的第二个参数
    * 这么写能够处理执行resolve遇到异常的情况
        + 如果resolve执行时遇到异常，将会条到`.catch`中


**Promise还有很多用法：all，race等**


### async、await

如果不喜欢.then这样的写法，可以使用async、await的写法

async的本质就是promise的语法糖，标记了async的function里面可以执行await的同步方法，await后的代码要等待await执行完成。

[猜]为什么promise能够做到同步呢？即.then().then()的同步原理
- 当执行第一个promise时，是异步操作，最后才能返回promise。所以如果promise没返回就不会执行下一个

```javascript
// callRing函数不变

// 有await这个关键词的代码只能在一个async 定义的函数里执行
async function action(){
    try{
        await callRing(true) // 如果是resolve则执行后续的代码
        console.log("resolve")
    } catch (e){
        // 如果reject则执行catch的代码
        console.log("resolve")
    }
}

action()
```

或使用IIFE的写法，定义成匿名函数然后立即执行

```javascript
// callRing函数不变

(async () => {
    try{
        await callRing(true) // 如果是resolve则执行后续的代码
        // 否则抛出异常
        console.log("resolve")
    } catch (e){
        // 如果reject则执行catch的代码
        console.log("resolve")
    }
})()
```


