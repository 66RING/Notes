---
title: 使用ftrace追踪内核函数调用
date: 2021-01-30
tags: 
- linux
- kernel
- tools
- ftrace
mathjax: true
---

# 1 初探ftrace遇到的问题

## 1.1 搞清楚ftrace是否标记出中断函数

根据[官网的描述](https://www.kernel.org/doc/html/latest/trace/ftrace.html#trace-options)，funcgraph-irqs选项，用于表明是否追踪中断。因此关闭后就不会记录中断信息。

```
funcgraph-irqs

    When disabled, functions that happen inside an interrupt will not be traced.
```


## 1.2 搞清楚为何两次ftrace开头不一样

由于ftrace是将数据保留在一个ring buffer中，当数据大于buffer容量时，旧的数据就会被丢弃。所有两次ftrace执行被丢弃的数据长度不同，导致看到两次的开头不同。

解决的方法很简单，通过`buffer_size_kb`将缓存开大即可。

```
echo 10000 > buffer_size_kb
```


# 2 使用ftrace追踪内核函数调用

> ftrace是一个Linux内核特性，它可以让你去跟踪Linux内核的函数调用。


## ftrace基本使用

通过操作`/sys/kernel/debug/trace`目录下的数据文件来使用ftrace。启动ftrace后ftrace会根据这些文件的内容执行对应的操作。

主要控制ftrace的文件如下：

- `trace`，查看ftrace追踪的结果
- `tracing_on`，控制ftrace的开关
    * `echo 1 > tracing_on`启动ftrace追踪
    * `echo 0 > tracing_on`停止ftrace追踪
- `current_tracer`，设置当前使用的追踪器
    * 可以在`available_tracers`中查看都支持哪些追踪器
    * `nop`追踪器：将nop写入`current_tracer`不追踪任何事件，并刷新trace文件
    * `function`追踪器：追踪函数调用
    * `function_graph`追踪器：类似`function`追踪器，但是能比较直观的查看函数的调用关系
    * [更多追踪器可以看这里](https://www.kernel.org/doc/html/latest/trace/ftrace.html#the-tracers)
- `set_ftrace_pid`，指定要追踪的进程的pid
- `set_ftrace_filter`，指定要追踪的函数名
    * `set_ftrace_notrace`，指定不要追踪的函数名
    * 支持`*`, `!`等通配符
- 以此类推，[详见](https://www.kernel.org/doc/html/latest/trace/ftrace.html#the-tracers)


ftrace还提供很多选项来自定义追踪的信息，主要在`trace_options`文件中。当需要关闭某个选项如`funcgraph-cpu`时，在其开头加上`no`写入即可：`echo nofuncgraph-cpu > trace_options`。

不同tracer下可以用的选项有所不同，但是大同小异，以`function_graph`tracer为例，有如下常用选项


- `function-fork`
    * 当打开时，`set_ftrace_pid`中进程fork产生的子进程的pid也会自动加入`set_ftrace_pid`中，当`set_ftrace_pid`中的进程退出时其中的PIDs会自动移除
- `funcgraph-irqs`
    * 关闭后，中断触发的函数将不会被追踪
- `funcgraph-duration`
    * 显示函数执行的时间，以毫秒为单位
- `funcgraph-proc`
    * 开启后将显示进程名，pid等信息
- [更多选项内容看这里](https://www.kernel.org/doc/html/latest/trace/ftrace.html#trace-options)

上述`trace_options`中的内容与`./options`文件夹中的数据文件对应


## 使用ftrace追踪一个进程的内核函数调用过程

刷新一下trace

```
echo nop > current_tracer
```

设置tracer追踪器，并设置一些选项：

```
echo function_graph > current_tracer        # 使用function_graph追踪器
echo nofuncgraph-irqs > trace_options       # 不跟踪中断
echo nofuncgraph-cpu > trace_options        # 不显示cpu信息
echo nofuncgraph-duration > trace_options   # 不显示执行时长
echo function-fork > trace_options          # 自动跟踪子进程
```

设置要追踪进程的pid

```
echo 14137 > set_ftrace_pid  # 跟踪14137号进程
```

追踪一段时间后关闭

```
echo 1 > tracing_on
# 一段时间后...
echo 0 > tracing_on
```

查看`trace`文件的内容，就是追踪的结果了

```
cat trace
```

需要注意的是，`trace`数据文件的大小有限，当超出大小限制后`trace`信息前端会被抛弃(因为数据保存在一个ring buffer中)。可以通过修改`buffer_size_kb`文件的值来设置每个cpu能够记录的缓存大小。

```
cat buffer_size_kb
```

每个cpu的缓存是可以单独设定的，如果cpu的设置的缓存大小不一致，`buffer_size_kb`中就会显示`X`

```
echo 10000 > per_cpu/cpu0/buffer_size_kb
echo 100 > per_cpu/cpu1/buffer_size_kb
```

`buffer_total_size_kb`会记录总缓存大小




