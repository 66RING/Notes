---
title: shell脚本杂技
date: 2000-01-01
tags: 
- operate
- shell
mathjax: true
---

## double dash `--`

单独使用double dash `--`一般用于表示命令行选项(command line options)的结尾，之后不再接收选项，只能接收位置参数(positional parameters)

如一下命令会查找`-v`而不让它作为命令行的选项

```sh
grep -- -v <file>
```

## 文件描述符与文件控制
 
- 文件描述符
    * https://zhuanlan.zhihu.com/p/109053744


## set

- `set -- `
- ...

https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html

