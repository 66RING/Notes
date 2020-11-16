---
title: curl基本使用
date: 2020-11-15
tags: linux, tools
---

等读完curl(如果有时间)，就改成curl all in one

curl是一个http**请求**工具

- `curl [options...] <url>`
    * `queryString`
    * `-s | --silent`，不显示进度条
    * `-o | --output`，指定请求结果输出到的文件
    * `-H | --header`，设置请求头
    * `-d | --data`，设置请求携带的数据
    * `-X | --request`，设置请求使用的方法

