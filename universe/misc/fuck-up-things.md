---
title: 天坑细节
author: 66RING
date: 2000-01-01
tags: 
- misc
mathjax: true
---

# 天坑细节

- 编译时, 对库使用的声明要放在文件后面, `gcc a.c -lcuda`
- 二分查找mid什么时候需要`+1`向上取整,
    * `[left, mid-1] [mid, right]`, `mid = (l+r+1)/2`, 因为我们要划分的区间是左小右大, 如果mid下取整，当落到右区间时mid == left大小不再缩小导致死循环
    * `[left, mid] [mid+1, right]`, `mid = (l+r)/2`

