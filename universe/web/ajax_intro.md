---
title: JQuery, fetch, axios
date: 2020-11-21
tags: web, ajax
---

## Ajax简介

Ajax本质就是用来进行异步的请求提交，那是怎么个异步法？

- 同步
    * 跳转或新开一个页面
- 异步
    * 页面局部变动，异步请求然后更新局部页面，而不更新整个页面

早期进行异步请求是通过XMLHttpRequest(xhr)，但是这种方法代码非常长，冗余严重。后来JQuery出现`$.ajax`，对xhr进行封装，又有了后来的fetch和axios。


## JQuery

```javascript
// 基本请求
$.ajax({
    method: "",
    headers: {"key": "value"},
    data: {},
    url: "",
    success: funciotn(){}
})

// 也可简写方法
$.get(url, data, callback)
$.post(url, data, callback)
```

*PS* :默认使用表单形式提交，需要提交json数据数据时，在headers中设置`content-type:application/json`，然后data中传入**json字符串**


## fetch

现代的浏览器中一般都会自带fetch方法，不需要下载任何js包

- 基本用法：`fetch("usl", {<args>})`
    * 返回一个promise对象

```javascript
fetch("url", {
    method: "",
    headers: {},
    body: "",
}).then(res=>{}).then(res=>{})
```


## axios

不单单可以在前端使用，还可以通过nodejs在后端使用

```javascript
axios({
    url: "",
    method: "",
    headers: {},
    data: {},
})
```

同fetch一样返回promise对象，可以通过`then()`进一步处理

*PS* :默认使用json形式提交，需要表单提交数据数据时，在headers中设置form的header，如`content-type:application/x-www-form-urlencoded`，然后data中传入这种形式(查询字符串)的数据`"a=123&b=321"`





