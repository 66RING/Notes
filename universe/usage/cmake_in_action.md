---
title: Modern Cmake
author: 66RING
date: 2021-05-22
tags: 
- cmake
mathjax: true
---


## basic TODO: rename

- [笔记内容来源](https://github.com/richardchien/modern-cmake-by-example)

经典三行

```cmake
cmake_minimum_required(VERSION 3.9)
project(answer)

# TODO: 
add_executable(answer main.cpp answer.cpp)
```

使用

```cmake
cmake -B build      # 生成构建目录，-B 指定生成的构建系统代码放在 build 目录
cmake --build build # 执行构建
./build/answer      # 运行 answer 程序
```

- `-B`指定生成的构建系统存放的目录
	* e.g. 生成make，ninja的系统
- `--build build`真正build，执行build目录中的构建系统
	* e.g. 用对应的make, ninja开始构建


## 解耦

### 拆分成库

制作静态库

```cmake
add_library(libanswer STATIC answer.cpp)
```

使用库: 给targe链接上库`target_link_libraries()`

```cmake
add_executable(answer main.cpp)
target_link_libraries(answer libanswer)
```

### 拆成子目录

> 进一步模块化

**头文件怎么办?**

TODO:


### 进一步解耦

细分同一层的子目录，`CMakeLists.txt`管理

```cmake
add_subdirectory(answer)
add_subdirectory(curl_wrapper)
add_subdirectory(wolfram)
```

## 使用第三方库

```cmake
find_package(CURL REQUIRED)
target_link_libraries(libanswer PRIVATE CURL::libcurl)
```

- `find_package(CURL REQUIRED)`
	* 找CURL库，`CURL`和`CURL::libcurl`这个名字是约定的名字
	* `REQUIRED`表示必要，没找到会报错
- `PRIVATE`
	* `libanswer`内部的内容，接口不暴露


## CACHE变量

"私密数据应该通过从外部传入"。Cmake中先"声明"

```cmake
set(WOLFRAM_APPID "" CACHE STRING "WolframAlpha APPID")
```

`set()`第一个参数是变量名，第二个参数是默认值，第三个参数 CACHE 表示是 cache 变量，第四个参数是变量类型，第五个参数是变量描述

`BOOL`类型的变量还可以用`option()`来设置

```cmake
set(ENABLE_CACHE OFF CACHE BOOL "Enable request cache")
option(ENABLE_CACHE "Enable request cache" OFF) # 和上面基本等价
```

**设置变量值** : 使用`-D`参数传递，或使用`ccmake`用TUI的形式修改

```cmake
cmake -B build -DWOLFRAM_APPID=xxx
```

程序中当作宏使用，e.g. 通过环境变量获取。

```cmake
target_compile_definitions(libanswer PRIVATE WOLFRAM_APPID="${WOLFRAM_APPID}")
```

注意什么时候需要引号，`WOLFRAM_APPID="${WOLFRAM_APPID}"`等价于`#define WOLFRAM_APPID "xxxxx"`。"原样替换"


## 设置的粒度

任何一个目录的CMakeList.txt里存在都会改变全局。

```cmake
set(CMAKE_CXX_STANDARD 11)
```

`target_compile_features`仅影响单个target，细粒度地指定feature

```cmake
target_compile_features(libanswer INTERFACE cxx_std_20)
```

## 单元测试

TODO:
