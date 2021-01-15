---
title: 使用systemtap实现进程的跟踪
date: 2021-01-13
tags: linux, systemtap, tool, kernel
mathjax: true
---

# 使用systemtap实现进程的跟踪

## systemtap安装

systemtap安装分为两个部分，一个是安装systemtap本身;另一个是安装对应的内核信息包(Kernel Information Packages)，否则systemtap将无法深入内核探测。

以下以在centos中安装为例。

- 安装systemtap
    * 使用对应的包管理工具安装，如centos中`yum install systemtap`
- 安装内核信息包
    * 需要注意的是安装的内核信息包需要与内核版本对应，如内核版本为4.18.0-147，就安装4.18.0-147的包
    * 使用`stap-prep`命名检查systemtap还需要哪些支持，一般需要安装如下几个信息包：
        + kernel-debuginfo
            * 可以在`http://debuginfo.centos.org/8/x86_64/Packages/kernel-debuginfo-4.18.0-147.el8.x86_64.rpm`中下载
            * 然后安装`rpm -ivh`
        + kernel-debuginfo-common
            + `http://debuginfo.centos.org/8/x86_64/Packages/kernel-debuginfo-common-x86_64-4.18.0-147.el8.x86_64.rpm`
        + kernel-devel


## systemtap基本使用

Systemtap的基本思路是监听事件(event)，并提供处理程序(handler)。当指定事件发生时，内核就会执行相应的处理程序。

Systemtap内置了很多事件，为用户提供了简单的API，以方便地编写处理程序。

语法类似C语言，探测事件的格式为：`probe event {statements}`。如我要探测哪个程序执行了内核中的`_do_fork`函数，可以这么写：

```stp
probe kernel.function("_do_fork@kernel/fork.c"){
    printf("%s: %s\n", execname(), ppfunc());
}
```

其中`kernel.function(PATTERN)`就是systemtap预置的一个事件，即当`PATTERN`匹配到的函数被调用是，触发对应的handler，这里是打印了调用的进程的名字`execname()`，和调用的函数名`ppfunc()`。PATTERN的格式是`func[@file] | func@file:linenumber`，支持正则表达式匹配。

除了`kernel.function()`事件外，有其他很多内置的事件[详见此处](https://sourceware.org/systemtap/SystemTap_Beginners_Guide/scripts.html#systemtapscript-events)。内置的函数如`execname()`等的使用，[详见此处](https://sourceware.org/systemtap/SystemTap_Beginners_Guide/systemtapscript-handler.html#syscall-open)。这里列出一些常用值

- probe，"探测"
    * `probe <probe-point> <statement>`
    * `probe-point`指定了probe动作的时机，一旦探测是事件触发了，则probe将从此处插入内核或进程
- API
    * `begin`, systemtap 会话开始
    * `end`, systemtap 会话结束
    * `kernel.function("sys_xxx")`, 系统调用xx的入口
    * `kernel.function("sys_xxx").return`, 系统调用xx的返回
    * `timer.ms(300)`, 每300毫秒的定时器
    * `timer.profile`, 每个CPU上周期触发的定时器
    * `process("a.out").function("foo*")`, a.out 中函数名前缀为foo的函数信息
    * `process("a.out").statement("*@main.c:200")`, a.out中文件main.c 200行处的状态
- 常用可打印值
    * `tid()`, 当前线程id
    * `pid()`, 当前进程id
    * `uid()`, 当前用户id
    * `execname()`, 当前进程名称
    * `cpu()`, 当前cpu编号
    * `gettimeofday_s()`, 秒时间戳
    * `get_cycles()`, 硬件周期计数器快照
    * `pp()`, 探测点事件名称
    * `ppfunc()`, 探测点触发的函数名称
    * `thread_indent(n)`，补充空格
    * `$$var`, 上下文中存在 $var，可以使用该变量
    * `[s]print_backtrace()`, 打印内核栈
        + 带s会返回堆栈字符串
    * `print_ubacktrace()`, 打印用户空间栈


### 小技巧

- 扫描所有探测点可用`stap -L|l`，如
    * `stap -L 'kernel.function(*fork*)'`，扫描所有包含fork的函数，并列出可用的变量
- 可以用`$`直接获取进程的全局变量，获取不到可以试试`@`
    * 如果变量不是本地变量可以这么引用`@var("varname@src/file.c")`
- 可以用`$1`，来接受命令行参数
    * `probe $1{}`
- systemtap中无论是指针还是结构体都使用`->`访问成员
- 输出整个结构体:变量结尾加一个或两个`$`
    * `$`后缀打印结构体
    * `$$`后缀打印结构体，如果结构体中包含复杂结构则将其展开
- 在return探测点可以用`&return`获取返回值，inline函数无法安装return探测点
- 使用`@cast()`来实现指针类型转换
    * `@cast($var, "new_type"[, "file_that_type_define"])`
- 同样使用`@cast`定义某个类型的变量，原理就是用`@cast`转换类型后赋值给一个变量
    * `c = &@cast($rev->data, "ngx_connection_t")`
- 对于多级指针如`**p`可以使用`[0]`来解引用
    * `$p[0]->id`
- 输出字符串指针
    * 用户态使用：`user_string`,`user_string_n`
    * 内核态使用：`kernel_string`,`kernel_string_n`,`user_string_quoted`
        + `user_string_quoted`是获取用户态传给内核的字符串，代码中一般有__user宏标记：
- 查看函数指针所指的函数名
    * 获取一个地址所对应的符号
    * 用户态使用`usymname(FUNC_PTR)`
    * 内核态使用`symname(FUNC_PTR)`
- 修改进程中的变量
    * `$var=value`
    * 需要注意的是stap要加`-g`参数才能修改变量的值
- 查看代码执行执行路径
    * `pp()`，输出当前被激活的探测点，即包括函数名、路径、行号等
- 关联数组
    * 关联数组必须是全局变量，用global声明。最多支持9项索引域
        ```stp
        global reads
        probe vfs.read {
            reads[execname(), pid()]++
        }
        probe timer.s(3) {
            foreach ([execname, pid] in reads) {
                printf("%s(%d) : %d \n", execname, pid, reads[execname, pid])
            }
            delete reads
        }
        ```
    * 可以使用`+`、`-`进行排序
        ```stp
        global reads
        probe vfs.read {
            reads[execname(), pid()]++
        }
        probe timer.s(3) {
            foreach ([execname, pid+] in reads) {
                printf("%s(%d) : %d \n", execname, pid, reads[execname, pid])
            }
            delete reads
        }
        ```
- `target()`，通过命令行`-x PID`传入id，作为target
- 对于指向基础类型的指针，以下函数可以获取其内核态的数据
    * `kernel_char(address)`
        + 从内核内存地址中获取char
    * `kernel_short(address)`
    * `kernel_int(address)`
    * `kernel_long(address)`
    * `kernel_string(address)`
    * `kernel_string_n(address)`
        + 获取的string长度为n bytes
- `$$var`获取探测点的所有变量
    * `$$locals`，`$$vars`的子集，仅包含本地变量
    * `$$parms`，`$$vars`的子集，仅包含函数参数变量
    * `$$return`，仅在return probe中有效


## 追踪进程

```c
void main(){
    for(int i=0;i<3;i++){
        fork();
        sleep(1);
    }
}
```

假设有这么一个程序，调用`fork()`函数然后`sleep`一秒，重复3遍。要想追踪它底层是怎么一步一步地调用的，首先需要对操作系统有所了解。这里提供一个思路：fork函数会底层会调用`_do_fork`，而sleep底层会调用`do_nanosleep`。

使用`systemtap`，扫描出大致的相关函数：

```sh
stap -l 'kernel.function("*fork*")'
```

得到如下输出，同理可以扫描`do_nanosleep`相关函数

```
$ stap -l 'kernel.function("*fork*")' 
kernel.function("__x64_sys_fork@kernel/fork.c:2304") 
kernel.function("__x64_sys_vfork@kernel/fork.c:2316") 
kernel.function("_do_fork@kernel/fork.c:2209") 
kernel.function("anon_vma_fork@mm/rmap.c:315") 
kernel.function("cgroup_can_fork@kernel/cgroup/cgroup.c:5804") 
...
```

为了进一步精确调用trace流，可以根据扫描得到的函数信息(如函数名，函数路径，代码行等)，在[LXR](https://lxr.missinglinkelectronics.com/linux)阅读源码进行进一步挖掘，然后设置探测点。

于是为了找出哪个程序调用了哪个函数就可以：

```stp
probe kernel.function("_do_fork@kernel/fork.c"){
    printf("%s: %s\n", execname(), ppfunc());
}

probe kernel.function("do_nanosleep@kernel/time/hrtimer.c:1678"){
    printf("%s: %s\n", execname(), ppfunc());
}
```


