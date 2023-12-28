---
title: Arch Linux 坑坑速查表
author: 66RING
date: 2023-01-01
tags: 
- archlinux
mathjax: true
---

# Arch Linux 坑坑速查表

## .Xauthority lock timeout

用`strace xauth list`可以看到操作具体流程, 主要就是`.Xauthority`可能是root用户所有了, 及其`.Xauthority-*`的冲突

删掉`.Xauthority-*`, 把`.Xauthority`改成本用户。删掉`.Xauthority`也可以试试

## N卡驱动

- 装完可以使用`mkinitcpio -P`手动更新内核
- lts要装lts的驱动



