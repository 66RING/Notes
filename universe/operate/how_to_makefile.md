---
title: How to makefile
date: 2020-2-26
tags: 
- makefile
- tools
- operate
---

# How to makefile

## cheat sheet

| f                          | 描述                      |
|----------------------------|---------------------------|
| `$@`                       | target                    |
| `$^`                       | 所有依赖，                |
| `$<`                       | 第一个依赖                |
| `=`                        | 整个makefile后再确定值    |
| `:=`                       | 那里用哪里展开            |
| `?=`                       | 如果没定义才赋值          |
| `$(wildcard PATTERN)`      | 获取所有匹配的文件        |
| `$(patsubst P1, P2, list)` | 读list,模式P1替换为P2     |
| `$(foreach var,list,text)` | 从list中取,放到var,做text |



## Compile process
How dose a project turn into an excutable file? There are four step:
- Preprocessing
    - Unfold all include libs, macro, head. And produce `.i` file.
    - (gcc/g++ commands)`-E`
- Compilation
    - The `.i` file produce an `.s` file(assembly code).
    - `-S`
- Assemble
    - The `.s` file produce an `.o` file(object file).It is a binary file.
    - `-c`
- Linking
    - Linking the `.o` file produce an excutable file.

Whole example
``` 
gcc -E halo.c -o halo.i  
gcc -S halo.i -o halo.s  
gcc -c halo.s -o halo.o  
gcc -o halo.o -o halo
```

## Why makefile
As we see, compile a file include so much process.Though `gcc halo.c -o halo` can do the same thing as above, you need to make sure all the `.c` file are in the directory. 
And what if a thousand file to compile?

## Lets start
### Format
``` makefile
<target>: [dependencies]
<tab> [command]

^ before command IS A <TAB>, and make sure a new line at the end
```
We should write "from rear to front". Which means write final target command first and then write dependencies commandS

### [to remove] Usage
``` sh
make target...
```

If didn't specify target, run the first target as default. That is why should we wirte main object at first.


#### Target


#### Prerequisites
#### Commands
#### Define

### Grammar

#### Echoing

make will print out every single command that will execute

use `@` to turn it off

#### Wildcard
#### Pattern matching
#### Variables

- `$`, `$$`
- `=`, `:=`, `?=`, `+=`

#### Implicit Variables

[handbook](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)

#### Automatic Variables

- `$@`
- `$<`
- `$?`
- `$^`
- `$*`

[handbook](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html)

#### Conditional
#### Functions


# CMake

Makefile may not work at different platform. Is quite annoying to rebuild a Makefile. So CMake is came.

CMake can auto generate Makefile, it is more cross-platform.

Usually, cmake command write in a `CMakeLists.txt` file.

## Some basic

``` 
CMAKE_MINIMUM_REQUIRED(VERSION define_a_minimum_version_here)

PROJECT(define_project_name)

AUX_SOURCE_DIRECTORY(/your/dir/source pkg_name)  # to packaging files in dir_source. So you do not need every single source file below.

ADD_EXECUTABLE(executalbe_name source_files...)
# or
ADD_EXECUTABLE(executalbe_name ${pkg_name})
```

## Multi Dir CMake

Every sub dir need a CMake file. So your main CMake file can use it.

``` 
AUX_SOURCE_DIRECTORY(/your/dir/source pkg_name)

# static library
ADD_LIBRARY(lib_name STATIC ${pkg_name})  

# dynamic library
ADD_LIBRARY(lib_name SHARED ${pkg_name})
```

Use sub dir

``` 
CMAKE_MINIMUM_REQUIRED(VERSION minimum_version)

PROJECT(project_name)

ADD_SUBDIRACTORY(./subdir)  # add subdir

AUX_SOURCE_DIRECTORY(/your/dir/source pkg_name)

ADD_EXECUTABLE(executalbe_name ${pkg_name})

TARGET_LINKLIBRARYS(executalbe_name SubLib)  # link sub lib
```


## Standard Project

📁/Project  
---📁/build  
---📁/src  
---📁/lib  
---📝/CMakeLists.txt  

- `./src/CMakeLists.txt`
    ``` 
    INCLUDE_DIRECTORY(./path/to/lib)  #INCLUDE_DIRECTORY(./lib)

    SET(EXECUTABLE_OUTPUT_PATH $(PROJECT_BINARARY_DIR)/bin)
    # set executable output to EXECUTABLE_OUTPUT_PATH/bin =../build/bin

    AUX_SOURCE_DIRECTORY(/your/dir/source pkg_name)

    ADD_EXECUTABLE(executalbe_name ${pkg_name})

    TARGET_LINKLIBRARYS(executalbe_name SubLib)  # link sub lib
    ```
- `./lib/CMakeLists.txt`
    ``` 
    AUX_SOURCE_DIRECTORY(/your/dir/source pkg_name)

    SET(LIBRARAY_OUTPUT_PATH $(PROJECT_BINARARY_DIR)/lib)
    # set library output to PROJECT_BINARARY_DIR/lib=../build/lib

    # static library
    ADD_LIBRARY(lib_name STATIC ${pkg_name})  

    # dynamic library
    ADD_LIBRARY(lib_name SHARED ${pkg_name})
    ```
- `./CMakeLists.txt`
    ``` 
    CMAKE_MINIMUM_REQUIRED(VERSION minimum_version)

    PROJECT(project_name)

    ADD_SUBDIRACTORY(./subdir)  # add subdir

    ADD_SUBDIRACTORY(./src)
    ```

- ccmake(config cmake)
- some other option
