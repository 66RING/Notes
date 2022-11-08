## stack

抛砖引玉

[TOC]

### 虚拟机做虚拟机的backend, 使用现有的公共抽象做跳板

- 公共资源/约定俗成的抽象做跳板
- `VM-pod`
- 分布式非容灾局部性调度
- shard化容灾方案

资源 => 无限, 无痛(无休了解硬件细节, 比如显卡这种闭源驱动)多虚一(究极自动换入换出)

问题在于CPU可以独立于内存存在么?

底层使用 **轻量VM**, 仅做一些资源的池化? 或者说是底层VM仅做特定资源的虚拟比如有的VM做内存, 有的VM做计算, 即不必是完整的一个虚拟机, 不妨称之为`VM-pod`

顶层VM再是全虚拟化的调度者

```
GUEST
------------------
VMM
-----NETWORK------
VMM0   VMM1   VMM2
----  -----  -----
HOST  HOST    HOST
```



### 目前的CUDA居然不支持利用磁盘的swapspace

也就是说hierarchy存在一部分缺失。这点是参考的hadoop file system: local memory -> rack(local disk) -> remote node。一个多层次多级别的解决方案。

即使不基于虚拟化, 这个多级的思路应该也是可以用到所有分布式系统中的。

2022-11-01


### 共振: 平板上一排摆钟, 板下是可以左右移动的滑轮

最后所有摆钟都会同步

2022-10-13 20:32


### 仿生: kcore与"多数人的表决". 能否利用"多数人的表决"的思路改进子图搜索算法

- 图计算中讲究寻找子图给我的感觉是类似与视线的聚焦
	* 尤其是kcore更像了, 外层是余光, 内层是聚焦
- 那是否存在另一种模型是"大多数的interset"驱动的

2022-09-30 10:30


### raft/paxos中"多数人的表决"启发子图算法

- kcore, clique是不是会比较难存在完美的符合条件的情况? 所以需要一些简化
- 那么kcore, clique中能不能不严格要求全链接或k链接, 找一个 **"intersection链接"优化子图问题**

2022-09-30 10:26


### 分布式一致性算法

- 如何制作"死锁"条件? 类似paxos和raft中的"多数人"
- kcore算法, 启发了一种图计算的思路
	* 不断删边, 最终k链接的会相互死锁得到结果
	* kcore是否存在的并行算法?

2022-09-30 10:23


### about ssd

ssd的写流程: 

- 拷贝块到内存
- **清空整块**
- 内存中修改, 整块写回

- 不能定点清除, write in page erase in block
- erase != delete, 有效数据写时才使用erase保证, 否则delete标记
- gc


2022-08-28

### 图计算引发的结构化的思考

我们说图计算的优势是它能反映出 "关系"。ok, 那现实中是否存在更多的信息需要抽象?

就比如, 结构化anything

e.g. textobject

2022-08-24 22:11


### 分形相似: 技术含量的图灵完毕

一个项目(object), 可能需要很多技术, 用到那么多技术是否就

e.g. 编译里涉及的很多内容

即使不完整那也可以object将互补完整

2022-08-24 22:05


### 树与局部性: 增量图分析

- 局部修改不影响外围: incremental parsing
- treesitter
	* incremental parsing
		+ 边type边parse
	* btrfs

2022-08-24 21:49


### 可扩展性问题

- 存在一个按需复制的人来stealing work
- 从而缓解瓶颈

2022-07-30


### 一步到位IPC

- 分布式 -> 多IPC
- 微内核 -> 多IPC
- 分布式微内核native一步到位?

2022-07-28


### RAID + 正交 + 码分复用

2022-07-21


### 逻辑数据分析

- 内核态只做少量的逻辑和涉及安全的操作
- offload内核态

2022-07-16

### distributed oriented DB

好像似乎都是要两段写才能保证安全, 无论分布式的还是，FS, DS的。那是否可以仅用分布式的log就行了，而不用DB本地的??distributed oranted db

本质是因为要恢复那得有原件

2022-07-13


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

- 数据集仓库
- 虚拟货币捐赠
- p2p

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


