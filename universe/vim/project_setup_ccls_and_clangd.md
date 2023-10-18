---
title: 给neovim配置项目信息
author: 66RING
date: 2022-09-22
tags: 
- vim
mathjax: true
---

# 给neovim配置项目信息

为什么要配置项目信息呢? `compile_commands.json`文件不行吗? 主要是为了刚开始编码时能有一些信息, 所以直接简单地手写一下`.ccls`文件或者`.clangd`文件会方便一点。

## clangd

> [clangd file](https://clangd.llvm.org/config)

`.clangd`文件由一个个片段(fragment)组成, 片段间可以使用`---`分割。本质就是解析成yaml, 所以配置是key value对的形式定义的。比如:

```yaml
CompileFlags:
  Add:
    - "-xc++"
    - "-I/abs/path/1/include"
    - "-I/abs/path/2/include"
  Compiler: clang++               # Change argv[0] of compile flags to `clang++`
  CompilationDatabase: "/path/to/compile_commands.json" # [Optional]
```

其中, 数组也可以写成`add: [-xc++, -Wall, -I/abs/path1, -I/abs/path1]`。需要注意的是路径要使用绝对路径[issue](https://github.com/clangd/clangd/issues/1038)

**官方文档怎么看**, 每个大标题就是一个fragment, 如`CompileFlags`。然后其中的小标题就是设置, 如`Add`, 会描述怎么设置

配置`.clang-format`文件可以控制格式化的格式, `.clang-tidy`可以控制代码检查等

## ccls

> [ccls-file](https://github.com/MaskRay/ccls/wiki/Project-Setup#ccls-file)

在项目根目录创建一个`.ccls`文件, 然后以行为单位添加编译器信息, 如`-I`, `-D`等。其中**第一行要是compiler driver**, 一般用`clang`, 后续每行添加一个参数(需要使用空格`-I foo`, 用`-Ifoo`)。每行开头可以使用一个或多个`%`开头的的特殊变量(详见官方文档)。

形如:

```
clang
%c -std=c11
%cpp -std=c++2a
%h %hpp --include=Global.h
-Iinc
-DMACRO
```

或者在`compile_commands.json`基础上追加, 第一行使用`%compile_commands.json`后不用再添加compiler driver

```
%compile_commands.json
%c -std=c11
%cpp -std=c++14
%c %cpp -pthread
%h %hpp --include=Global.h
-Iinc
```



