---
title: vue速成
date: 2020-4-26
tags: 
- web
mathjax: true
---

# 模板语法

## 插值

- "Mustache"语法(双大括弧): `{{}}`
- 插入纯文本
- 插入html可以用`v-html`

```vue
<p>Using mustaches: {{ rawHtml }}</p>
<p>Using v-html directive: <span v-html="rawHtml"></span></p>
```

## Attribute

绑定属性与数据

- `v-bind:`
	* 绑定属性与数据如id属性`v-bind:id="dynamicId"`
- `v-bind:[attribute]=""`
	* 动态参数
- `v-on:`
	* 监听DOM事件
	* 如`v-on:click`，`v-on:submit`
	* 可以添加修饰符，如`v-on:submit.prevent`
- `v-if=""`
	* 条件
- `v-for="<i> in <list>"`
	* 循环
- 缩写
	* `v-bind`缩写：`:`，如`:href`, `:id`
	* `v-on`缩写：`@`，如`@click`, `@[event]`


# Data Property

`data()`函数，返回一个对象，vue以`$data`存储在组件中

```
data() {
	return {id: 4}
}
```

# UI框架(ant design)组件使用

## 表单(登陆框等)

表单使用[FormModel](https://www.antdv.com/components/form-model-cn/)，表单用`<a-form-model>`表单中的每项用`<a-form-model-item>`，用`v-model`进行表单数据绑定，其他属性详见API描述

- `v-model`
	* 与表单数据绑定
- `ref`
	* TODO

## layout

<a href="https://www.antdv.com/components/layout-cn/">Layout布局</a>


## router

- `router.push`
- `router.path`
- `next()`



## menu


