## stack

### 数据文件cache未初始化0，导致历史敏感记录泄露?

### 设计

- bspwm = QEMU HMP/QMP
	* cool


### VMM yield insersion

qemu多线程模型，在vm exit进入到qemu执行模拟时，能不能给当前需要的模拟的CPU注入一个"yield中断"，然后让该vCPU先执行其他进程呢？


### 编译器做手脚问题

编译器是可以偷偷做手脚的


### bypass virtualization

现在很多要求高性能的软件(如数据)都会绕开操作系统，自己定制更高效更稳妥的解决方案。

ok，那如果这个操作系统是建立在虚拟环境上的呢？能不能再bypass掉将性能发挥到极致？

vfio?

### 快照base测试机

比如我要跑个压力测试，重复很多遍，但是我可能需要一边测试一边修改代码，不希望修改的代码对测试造成影响。

### 有限元 局部性 宏观

### 基于共振的分布式一致性方案

1. 相同的目标为催化剂

### 序列

混沌 = 序列?

### stack背后的究极秘密是什么

> stack can reverse

- 为什么"先构建后释放"是那么的自然，就像先打基础，最后再把公共的基础(ref最高)"释放"
- stack base virtual machine


### 挖矿别因式分解了，算模型吧

- 算力用在有价值的地方
- 那该如何量化和评判呢？


### 全新的交互逻辑

逻辑类似对象方法总是返回对象本身，也类似pipe的哲学。一个app这么设计好像没什么，what if一系列一生态？就像很多unix程序

```
fn setX() Self {
	Self
}
obj.setX().setY()
```


### 傅立叶级数在人工智能领域的应用

**正交**

核磁共振


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


