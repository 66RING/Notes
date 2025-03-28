## stack

抛砖引玉

[TOC]

### The auto-regressive OS

llm serving可以像操作系统一样。本质就是pc指针沿着sequence len方向去执行。

- prefill阶段, pc只管step
- decoding阶段, 生成logits, 然后pc step
- schedule by redix tree

- 模型(CPU)插槽
- 内存(MEM)插槽


2024-10-14 14:19

### kernel bypass LLM

cuda申请内存需要大量访问驱动，进行大量内核态和用户态的切换。如果能在用户态实现一个用户态的显存分配器(缓存分配器)就能实现加速。

不过可能cudaMalloc已经有了, 需要cudaMalloc, cudaFree测试一下。

好吧, pytorch已经实现了: `cuda.empty_cache()`

2024-01-05 16:58


### distributed attention + best match scala

balance attention在一定的grid下表现最好，那么将attention做分布式处理，分割成性能最好的小块执行。

2023-12-25 10:48

### balance attention + hierarchy aware

2023-12-25 10:48

### 当前序列并行的问题

都是对整个序列进行均分的, 但其实可以对部分均分。因为对于一个设备来说, 部分seq不在本地, 在其他设备或者在CPU都是一样的!

2023-12-05 10:32

### 结构化的attention

类似treesitter, 将prompt结构化

2023-10-29 21:47


### 有状态的生成式AI

transformer自回归, 其"状态"就是输入序列, 但这样的状态表示好像太过直接? QKV的能不能随着状态动态变化一点?

2023-10-09 15:17


### 基于提示词工程的上下文回溯AI

> 假设: 用k各单词就能够描述清楚一件事情。k也许是3k, 4k

- 一件大事情由若干小事情组成, 分别描述清楚小事情后大事情也就描述清楚了
- 使用k各单词就足以描述清楚小事情
- 思路类似函数调用栈, 描述完小事情后回溯, 恢复上下文, 再根据恢复后的上下文描述下一件事情

```cpp
prompt_l1_1()
    prompt_l2_1()
    prompt_l2_2()
prompt_l1_2()
    prompt_l2_3()
prompt_l1_3()
    prompt_l2_4()
```

那生成顺序be like:

1. `generate({prev, curr, next})`, 主要生成curr, 但会感知prev和next以丝滑过度
2. `generate({prompt_l1_1, prompt_l2_1, prompt_l2_2})`
3. `generate({prompt_l2_1, prompt_l2_2, prompt_l1_2})`
4. 其中`prompt_xx_x`这些提示词可以**使用外部程序保存到其他地方, 如磁盘**
5. ...

2023-09-19 17:29


### 分块生成式AI

> 函数调用和返回

- 预测下一个词的抽象是不是欠妥, 应该是预测下一个模块, 然后模块内展开, 然后记忆可以回溯到high level再处理下一个模块
    * **AI懂得抽象吗?**
    * 抽象, 展开, 抽象, 展开
    * aka, 分模块的生成式AI

2023-09-06 10:55


### 为什么语言模型ChatGPT会产生让人震惊的涌现?

世界是连续的, 预测下一个词的本质其实是一种类似**拉普拉斯兽**一样的根据时间空间对于下一状态的预测

所以不单单预测下一个词会有这种涌现, 预测"下一个"声音, 像素, all!

所以根据拉普拉斯兽的理论, 是否可以引入其他数据用"多模态"来碾压某个具体的AI任务 

2023-08-23 12:28


### 神经模板和特化

仿生生物组织的分化

2023-08-07 23:42


### 中文房哈希

将一些内容抽象成一些词"hash", 缩短prompt的长度, 从而能够理解和记忆更长的内容

2023-08-07 23:42


### 智能体定理

> 66RING猜想

$$
Q_k = C_k \times b
$$

- b: 基础智能体逻辑熵
- $C_k$, 基础智能体到介质k的转换常数
- $Q_k$: 目标智能体逻辑熵

理论说明可以通过枚举的方式模拟任何智能体, 需要的规则数其实不需要大于b

e.g.AI与人做对比, 人智就是b, 人脑到计算机(介质k)的转换常数C, 所以AI的开销为Q

2023-06-22 10:36


### 开门造车APP

> 系统创造的本质就是枚举各种优化, 局部最优组合出整体较优

- 每个人向玩梗一样的post出自己的idea, 可以认真可以玩梗
- 跟帖的主要都是"狗头军师", 大家在比较和谐的氛围内头脑风暴
- 一来吸引志同道合的人, 二来给小孤独出点注意
- **TODO: 参加hackathon**

e.g. 小孤独约会计划

```
L1: 带去XX玩, ...
L2: 直男才会带去XX玩
```

2023-06-22 10:18


### 混沌交给编译器

- **混沌交给编译器自动生成, 但面向人的接口仍是分层抽象的**, 从而缓解分层带来的性能问题
- 某种编译器友好的, 分层层面的IR表示

2023-06-22 10:01


### 分布式计算图优化

- **分布式图优化 + 图分割**
    * 但可能不是那么well motivated, 但是我觉得有些东西不是那么straight forward的, 比如GPT这种原理
    * 一个motivated可以是方便研究员做研究
- 异构分布式 + 图优化: A100 + V100
- 基于图算法的搜索空间优化
    * 关系引入了**隐秘信道**
    
2023-06-21


### 图计算公开课

- 图计算公开课 -> 解决一个实际的ml问题(metaflow)
- taccl就是想把图优化应用到集合通信上吧，只不过只有拓扑没有图, 所以就定了个所谓的通信草图然后在草图上优化

2023-06-21


### 虚拟tensor智能体

> 参考虚拟内存

- 最小tensor单元, 类似页表, (编译器自动, 缓存友好)
- 3D并行切分不均回导致气泡变大的
    * tensoer太小也切??

2023-06-19


### 全仿生AI

- 人脑神经元高度并行, 虽然有点延迟
    * 部分是异步的, 部分是同步的
- 激素: 类似ResNet的本质
    * 某种保证下限的东西
- 周期激素和应激激素
    * 即使调整和代谢调整
- 有没有可能brain每个神经元原本是一样的, 只是收到不懂训练才分化出不同功能。不然怎么解释其强大的适应力
    * 所以需要一个归一化的过程， 这样大家看到的东西都是同类了

2023-06-17


### AI只拟合"人智"不够

根据分形相似, 没准可以利用AI的这个原理拟合万物，包括推理、预言等。

2023-05-10 22:20


### ml-sys革命

感觉ml-sys里面的东西都比较好想到啊，数据并行，模型并行，混合并行等。但是为什么算进几年才出现呢？感觉一个原因算transformer开启啦大模型时代，另一个原因是**烧钱**。系统的的文章实现可以那么多可能算因为便宜吧。所以可不可以开发一个模型系统作为baseline, 降低ml-sys的成本从而推动发展？

2023-05-10 12:56


### 代理server

> ZeRO + parameter server

- 模型小时传模型
- 数据小时传数据

2023-05-09 12:09


### 计算图与OS内核与程序合成

2023-05-03 23:02

### 连续系统的智能涌现

- 生命游戏: 只关注邻居细胞
- ChatGPT: 下一个词的概率判断 + 一定范围内的上下文
- 拉普拉斯兽: 每一个时刻的所有状态都能精准计算就能推理出未来

似乎可以多考虑不起眼的"连续系统"

2023-05-02


### "超并行"和整体元胞

整体看成(编码成)元胞, 只受到邻居影响(连续性)，且实时性不敏感。毕竟人的神经元也不是量子纠缠的吧。

2023-05-01


### 编码is all you need

举例子

- 语言是一种编码
- 编译是一种编码: char -编码-> token -编码-> 语法
- 其他未知编码

2023-05-01


### 自举的AI程序

就是说AI程序的代码也是AI自动填充且动态变化的。需要用上OS, Arch, Compiler等知识。

为了防止自举的出来的程序引发主程序崩溃可以参考微内核的设计。

2023-04-11 23:55


### 一种新的关系抽象

我不太清楚最先进的AI中是怎么抽象联系/关系的，直接是一个固定的weight作为参数？如果的固定的weight的话那应该可以变成动态的weight。

应该加点加速度和初速度之类的考量。比如人与人之间的知识共享，如果一方输出越说越多过于强大会导致别人不敢发表自己的想法，所以需要到一种刚好"混沌"的状态。

2023-03-15

### 自然最通晓

2023-01-28

### page table and extendible hashing

申请内存需要申请pte对吧, 然后我们又是lazy的page demand对吧, 那这个page demand的剩下的开销就是申请pte的开销了。512GB大概消耗1GB pte, 这部分开销需不需要减少呢？

比如一级页表只看高位pnn就可以看到有没有必要继续申请后续的pte了。

也就是说连pte的申请我们也lazy。

所以可以不可以和extendible hasing结合

2022-12-30 14:31


### chaos & 熵增

- 复杂指令集, 精简指令集 => 混合模式
- OLAP, OLTP => HTAP
- 宏内核, 微内核/外核 => 多核
- 共有云, 私有云 => 混合云
- ... 动态更新中

2022-12-29 22:59


### 层次结构inspect

```
???
--------------------
cloud: azure, aws, ...
--------------------
cloud resource manager   <-  如果有个open interface, 那就可以由第三方来做sky computing了
--------------------
vm/container
--------------------
api
--------------------
syscall/abi: linux, windows
--------------------
isa: x86, arm
--------------------
bare metal
```

2022-12-29 22:56


### sky computing

云计算, cloud依赖强, 但是连通性不高

sky computing

[Xline](https://github.com/datenlord/Xline), 就是这么一个多数据中心的场景的分布式kv

2022-12-12 13:23

### LSM的启发

LSM适合写多读少的场景

ok, 读写不能双全

那有没有某种适合读多写少的设计呢? b plus tree就可以了吗

2022-12-02 10:12


### GPU加速的图数据库

图结构(关系)可能和传统比"大小"的方式有所不同?

2022-11-26


### 传播算法与生命游戏

2022-11-17

### work stealing

数据迁移不如上下文迁移

2022-11-16


### 虚拟机做虚拟机的backend, 使用现有的公共抽象做跳板

- 公共资源/约定俗成的抽象做跳板
- `VM-pod`
- 分布式非容灾局部性调度
- shard化容灾方案
- 懒加载, 增量启动

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

2022-11-10


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

### 开门造车APP

> 系统创造的本质就是枚举各种优化, 局部最优组合出整体较优

- 每个人向玩梗一样的post出自己的idea, 可以认真可以玩梗
- 跟帖的主要都是"狗头军师", 大家在比较和谐的氛围内头脑风暴
- 一来吸引志同道合的人, 二来给小孤独出点注意
- **TODO: 参加hackathon**

e.g. 小孤独约会计划

```
L1: 带去XX玩, ...
L2: 直男才会带去XX玩
```

2023-06-22 10:18

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

### team -lorde

Not very pretty, but we sure know how to run things.

### 敢于开第一枪

开了第一枪才给后续很多枪做参考

### 1

one get bored, some bit change, humanity rebuild, the lucky one lucky enough, and enjoying this new life. However not everyone so luck.

### [Sisyphus and the Impossible Dream](https://www.youtube.com/watch?v=9IiTdSnmS7E)

What matters are the experiences, the journey

Life is a battle and there times when you need to accept that you've been beat.

it's been a long road and there's a lot to be proud of.


