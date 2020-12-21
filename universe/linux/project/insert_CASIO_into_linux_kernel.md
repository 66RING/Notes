---
title: 如何向linux内核插入新的调度器
date: 2020-12-07
tags: linux, kernel
mathjax: true
---

## 如何向linux内核插入新的调度方法

这里演示以下如下向linux内核中插入一个线程的调度器：CASIO([源码](www.TODO.com)) 

TODO rebuild **注意** 现代版本的linux内核(4.18)中调度器的入口不再是`./kernel/sched.c`，而是在`core.c`，而调度类分离成`rt.c`，`idle.c`，`fair.c`，`stop_task.c`，`deadline.c`，对应5个调度器类


### linux进程调度机制

- TODO 
    * [ ] building...
    * [ ] 需要重新整理
        - task重新描述一下
    * WHAT IS GENERIC SCHEDULER AND CORE SCHEDULER.

- 触发调度的时间
    * 1. 进程休眠或者因为某些原因让出CPU
    * 2. 分时机制定期检查触发进程调度
- 两个调度器，generic scheduler和core scheduler
    * generic scheduler
        + generic scheduler起调度员的作用，负责选取调度类和底层的上下文切换
        + generic调度器只器分配作用，不会参与进程的调度，而是完全交给进程的调度类处理
    * core scheduler
        + 核心调度器用来管理运行队列中的活跃的任务，每个CPU都有一个自己的运行队列
    * 调度器不会直接操作进程，而是控制**调度实体**
        + 一个调度实体是`sched_entity`的一个实例
        + 这样不单可以调度一个进程，还可以调度更大的抽象，如实现组调度(CPU时间先在组间分配，再在组内分配)
- 6种调度策略
    * 用于对不同类型的进程进行调度
        + 比如`SCHED_NORMAL`和`SCHED_BATCH`调度普通的非实时进程, `SCHED_FIFO`和`SCHED_RR`和`SCHED_DEADLINE`则采用不同的调度策略调度实时进程, `SCHED_IDLE`则在系统空闲时调用idle进程
- 5个调度器类


#### 运行队列

每个CPU都会有单独的一个`run_queue`，核心调度器`core scheduler`用它来管理运行中的进程。`struct rq`有如下重要结构

- `nr_running`，可运行进程的数量
- TODO


#### 调度类

- 调度类决定下一个task是什么
    * 内核提供了很多调度策略，而调度类就是以模块化的方式对这些调度策略的实现，调度类之间彼此互不打扰
    * 每个task都有自己的调度类，而调度类就负责管理它的
- `sched_class`结构体中有许多函数指针，主要内容有
    * `enqueue_task`，新增一个进程到运行队列(这里所谓是队列是一种抽象，可以是更复杂的结构，如CFS的红黑树)
        + 时机：进程从`sleep`装态到`runnable`状态时
        + 当进程在运行队列中注册后，其调度实体中`on_rq`字段置1
    * `dequeue_task`，将一个进程出运行队列
        + 时机：进程从`runnable`装态到`un-runnable`状态时，或者由内核引起(如优先级改变时)
    * `sched_yield`，主动触发调度
    * `check_preempt_curr`，如果必要的话可以用新唤起的进程抢占当前进程
    * `pick_next_task`，让CPU执行一个任务(进程)
        + 时机：`put_prev_task`后，当前任务别切换前


#### 调度实体

调度器通过调度实体来操作一个更大的抽象，而不局限于调度进程。有如下重要结构

- `load`，定义每个调度实体的权值，每个调度实体的权值又会影响到运行队列的总权值
- `run_node`，顾名思义，采用红黑树存储每个调度实体
- `on_rq`，指明当前调度的实体是否在运行队列中
- `sum_exec_runtime`，记录进程使用CPU的时间
- `update_curr`，用于计算`sum_exec_runtime`
    * 用当前时间减去`exec_start`后将时间间隔加到sum中，然后用当前时间替换`exec_start`，开始新的循环
- `vruntime`，用于记录虚拟时间
- `prev_exec_runtime`，当进程让出CPU后，当前的`sum_exec_runtime`会存到`prev_exec_runtime`中
    * `prev_exec_runtime`，会在之后抢占中用到
    * 当然`sum_exec_runtime`仍然保持，没有重置

因为`task_struct`中包含`sched_entity`的实例，所以一个task是一个调度实体，这句话反过来就不成立。


#### 优先级

TODO pass 还没详细进行

`task_struct`优先级由三个元素表示：`prio`和`normal_prio`表示进程的动态优先级，`static_prio`表示静态优先级。`rt_priority`表示实时进程的优先级。静态优先级在程序运行时就分配了，它可以通过系统调用：`nice`和`sched_setscheduler`修改。

`normal_priority`基于静态优先级和进程的调度策略，因此不同静态优先级会导致进程的不同`normal_priority`。如实时进程和非实时进程有别。


### 写入内核

- 改动(都用CASIO标注)
    * `/kernel/sched/sched.h`

#### 配置文件

在配置文件`.config`中把CASIO调度打开，因为很多地方，如调度器入口文件，需要根据宏`CONFIG_SCHED_CASIO_POLICY`进行条件编译。在`Kconfig`中加入CASIO调度器选项

```
# .config

#
# CASIO scheduler
#
CONFIG_SCHED_CASIO_POLICY=y
```

```
# Kconfig

menu "CASIO scheduler"

config SCHED_CASIO_POLICY
	bool "CASIO scheduling policy"
	default y
endmenu
```


#### 准备头文件

准备头文件`sched.h`，为CASIO正常运行做准备，见patch文件`TODO上传patch文件`

对于调度类的优先级，可以在头文件`sched.h`中将最高优先级的调度器类`sched_class_highest`设为CASIO调度器类。在4.18的内核中默认的`sched_class_highest`会根据SMP(对称对处理)配置`CONFIG_SMP`决定

```c
#ifdef	CONFIG_SCHED_CASIO_POLICY
	#define sched_class_highest (&casio_sched_class)
#else
	// statements like #define sched_class_highest (&rt_sched_class)
#endif
```

所属进程的优先级顺序为

```
stop_sched_class -> dl_sched_class -> rt_sched_class -> fair_sched_class -> idle_sched_class
```

或者，CASIO调度类next是`rt_sched_class`，修改`dl_sched_class`的next为CASIO也是可以的

TODO PASS


#### 修改逻辑

CASIO调度类next是`rt_sched_class`，修改`dl_sched_class`的next为CASIO也是可以的

TODO


## 问题

### 问题一

现在内核的结构变化很大，有些原本直接写在头文件的东西，现在可能仅仅在一小块作用域里。当然我可以无视条件编译直接定义，但是还是规范好吧随便加到个头文件欠妥，因为这个这个内核以后可能还要用来实验。

CASIO需要这样一个结构，但是新版的内核中不再在头文件中定义这个。当然我仍可以在头文件中写这个，但是为了同一规范，我也应该把它定义在一个特定作用域中，应该在哪呢？这个是结构是干什么的呢？

```
 struct sched_param {
 	int sched_priority;
+
+#ifdef	CONFIG_SCHED_CASIO_POLICY
+	unsigned int	casio_id;
+	unsigned long long deadline;
+#endif
 };
```



