## stack

### 基于声明周期和两段锁协议的statically analyzes

许多操作应该是一个不可分的事务，但程序员在加锁的时候可能会疏忽，没有遵守两段锁从而导致并发bug。


### 是否存在基于向量/矩阵的分布式方案?

- 类似RAID5
- 也许可以牺牲一点safty，但是是大概率不会出现safty问题
- 这么做的优势又是什么？

### 负载均衡与公平调度

- 可能会想，一个cpu上有多个任务，若其他cpu空闲，就可以发任务发出去
	* 如果空闲cpu的大量工作刚好有唤醒了，那又得重新均衡，反复迁移多浪费
	* 再考虑numa的情况，又该如何优化开销?

### 混沌流水线

混沌流水线，一种抽象，类似管程这种形式的抽象


### 内存共振

我们不能预测未来，所以不能选择什么换出。

但我们能不能引入"外部共振"，就像随机换出策略在大量数据的情况下居然可以优于LRU



## business

### Fault-tolerant QEMU

### Dshub

- metalink
    * http
    * bt/pt?
    * ftp
- bit coin reward
- code support


### Auto rise script


### 学生比赛前端模板

### 基于docker的云平台

- 内网通信优化(软件源、ssh等)
- 外网访问
- ssh


### 捧臭脚业务

- 演员捧臭脚，满足他人优越感和比较的心理

### 无情的玩梗机

### 如何做一个终端模拟器


### 复读机式备忘录

### 做个backtrace工具

- 维护记录打开树
- 退回
	* 仅历史退回
- 前进
	* 对处理函数闭包，可以利用lsp
	* 具体目标前进，如果是新开，加入维护树
	* 历史前进，如果打开树只有1，且要开的和打开的一样，直接跳
- 打印记录树，并提供树中任意文件跳转

### 时间管理系统/计划表

- 时间块 10m/b
	* 一天减去睡觉 `(24-7)*60 / 10 = 102`
- 常用模型

### ~~unikernel -> cloud provider?~~

go bring in gorotine to boot up development of distributed system. It provide go gorotine as base tool.

what if 

- a unikernel, provide the base tools for single purpose OS
- we can be developer of that basic tool
- and we can be the cloud provider
- even in the internet of things

## BOOK

- 沉默螺旋理论

- 银杏书
	* https://ipads.se.sjtu.edu.cn/mospi/
- ostep



## 算法

### KMP

各个子串的"公共前后缀"，下次匹配位置=右移子串长+左移公共前后缀长

## CS

### 负载均衡与公平调度

- 可能会想，一个cpu上有多个任务，若其他cpu空闲，就可以发任务发出去
	* 如果空闲cpu的大量工作刚好有唤醒了，那又得重新均衡，反复迁移多浪费
	* 再考虑numa的情况，又该如何优化开销?

### 混沌流水线

混沌流水线，一种抽象，类似管程这种形式的抽象


### 内存共振

我们不能预测未来，所以不能选择什么换出。

但我们能不能引入"外部共振"，就像随机换出策略在大量数据的情况下居然可以优于LRU








































## MOVIE

### 1

one get bored, some bit change, humanity rebuild, the lucky one lucky enough, and enjoying this new life. However not everyone so luck.


