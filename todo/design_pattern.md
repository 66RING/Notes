---
title: 设计模式及其case
author: 66RING
date: 2023-08-27
tags: 
- misc
mathjax: true
---

# 设计模式cheat sheet

## 单例模式(singleton)

维护一个全局对象, 只在第一次访问时构造它, 后续不再构造直接使用

- case
    * 一个全局的链接池


## 工厂模式(factory)

根据必要信息直接返回一个对象, 你不用关心细节

- case
    * 根据公司场景不同会连接到不同后端, 比如很多要用trait/基类/接口的构造场景


## 抽象工厂模式(abstract factory)

工厂的工厂

- case
    * 用了工厂模式后发现创建工厂本身需要很多case要处理时
    * `create_factory(factory_id).create_product(product_id)`


## 创造者模式(builder)

一个对象需要很多参数构造时, 可以考虑一点一点构造的方式, 如果`builder().set_name(name).set_age().build()`

- case
    * 命令行参数的解析, 我们可能会考虑很多flag, 通过一点一点`set_xxx()`构造来添加我们的考虑


## 原型模式(prototype)

构造一个对象的成本比较高时, 我们可以从缓存中的一个现成对象直接克隆

- case
    * fork系统调用


## 适配器模式(adapter)

本质就是oop中的接口/trait/基类, 但是接口的接口

- case
    * 多个对象都实现一种接口(adapter), 然后可以用这个接口来方便的调用


## 装饰器模式(decorator)

接口的接口, 包装原本类在保证原本类功能的前提下提供新的功能

- case
    * 一个3DObject存在shape, color几个相互独立的领域


## 代理模式(proxy)

TODO: https://zhuanlan.zhihu.com/p/128145128

















