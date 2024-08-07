---
title: miniob项目笔记
author: 66RING
date: 2000-01-01
tags: 
- database
mathjax: true
---

# miniob项目笔记

## 读码tips

- 可以先看结构体猜人家怎么实现


## 目录结构

- `src/observer/sql`, 内部文件夹表示各个模块



## 文件管理

固定存储目录: `db/sys`

- 表文件(元数据): 命名规则`<表名>.table`
- 数据文件: 命名规则`<表名>.data`
- 索引文件: 命名规则`<表名>-<所创建的索引名>.index`


## 内存管理(Buffer pool)

> 类似CMU15-445

- DiskBufferPool
    * BPFrameManager: 管理内存中的page(注意和操作系统page区别，这里的page对应磁盘中的数据)
        + ⭐整个miniOB中只有一个BPFrameManager对象
        + Frame与page对应, 可以通过`file + page num`索引到
    * Replacer: 
        + lru, clock
    * BufferPool结构: 类似FAT, **一个BufferPool对象对应一个物理文件**
        + 若干page(相当于基本单位), 每页的内存布局都是: `|页号|数据|`
            + 数据部分内容由具体使用BufferPool的结构确定
        + 第一页: PageHeader, 内存布局
            + `|页号|当前文件总页数|已分配页数|bit map|`
            + TIPS/TODO: 文件大小受header限制, 可以使用多级索引的方式做扩展, 类似文件系统


## 记录管理(Record Manager)

> log

与BP交互, 记录存放到页中。目前是定长的record


## drop table实现为例

整体流程(看源码时就可以针对得看):

```
parser -> plan cache -> resovler -> transfomer -> optimizer -> executor
```


1. 先参考Table::create的流程
    1. `ParseStage::handle_request()`处理请求, 解析出命令, 打上标记
    2. `ResovleStage::handle_event()`进一步解析成语句`stmt`
    3. 我们仅需要关注执行阶段的内容, 会在`handle_request()`中根据不同case处理
        1. 获取建表上下文(用户命令内容), 获取当前数据库实例, 对当前数据库建表: 根据表名和文件格式协议找到对应文件, 根据上下文建表
        2. 创建表元数据, 
            - 元数据怎么创, 写到哪? 直接传给`Table::create(path)`就是meta data, 即根据这个path读写
        3. 创建表数据: 申请page, 写入header信息, 写回文件
        4. 创建记录文件
        5. 将所创建的表放到`opend_table`中, miniob初始化时会将所有表保存到`opend_table`中
2. 实现`Table::drop`
    1. 获取上下文, 找到数据库, 对数据库删表
    2. 删除索引, 记录文件, 表数据文件, 元数据文件, 与创建顺序相反
    3. 释放内存资源
    4. 从`opened_tables_`中移除


TODO: 搞清楚bpm和DiskBufferPool的关系

- `create_file`仅是文件的创建, 不实际添加到bpm中管理
- `open_file`添加到bpm中管理


## resolver

怎么解析

```cpp
RC Stmt::create_stmt(Db *db, const Query &query, Stmt *&stmt);
```


## 字段解析

> int, float等类型的字段, 需要增加日期类型字段的支持

- `parse_defs.h`定义词法分析语法分析中用到的结构
- 词法分析
    * 修改`src/observer/sql/parser/lex_sql.l`, 新增字段类型
- 语法分析
    * 修改`src/observer/sql/parser/yacc_sql.y`, 新增类型的处理
- resolver
    * `query`转`statement`
    * 对转换进行扩展, 使它能够支持日期字段: 字符串转日期, 内部日期如何表示(e.g. 用一个整数/时间戳)
        + `sscanf`, 日期合法性(润年)
- 为我们的类型实现`comparator`


## 线程池

> SEDA框架

一个线程池中有多种Stage, Stage之间通过时间进行通信。不同类型的也会会分配到不同的线程池, 降低相互干扰。

举个例子: miniob中SessionStage, ParseStage, ResolveStage, ExecuteStage都在一个SQLThreads线程池中。用户发送请求到服务端, 服务端就会事件通知SessionStage, 然后事件层层传递。

`observer.ini`中可以看到有几个线程池, 配置线程池的个数, 配置每个Stage所处的线程池, Stage关联的下一个Stage等。

- 初始化: `prepare_init_sedo`
    * `StageFactory`
- 例子
    * `SessionStage::make_stage`


## OceanBase进阶

### 存储引擎结构: LSM tree

TODO: 学LSM tree, 学机制够了么?

LSM tree的核心: write ahead log, 但是的分批的(SSTable)

bp tree vs lsm tree:

- bp tree: 原地更新 + 定长块
- lsm tree: append only + 不定长
- 因为定长块的存在, bp tree的压缩和解压缩就比较麻烦(解出来大了怎么办)。lsm tree就比较自然

- 写放大: 一个写入可能会触发多个压缩导致写入放大
- 读放大: 读取数据可能要扫描多个sstable, 扫描会多很多
- 空间放大: 因为append only, 历史数据不会立刻清理


#### 各种压缩策略

- Classic leveled compaction
    * N个level, 每个level一个SSTable, level间的SSTable往往成倍数关系(fan out, 扇出)
    * 当一个level到达数据量阈值时触发压缩压缩到下一个level
    * Li层数据与Li+1层数据在row key有交集的地方进行压缩, 得到新的Li+1层数据
    * 雪崩效应: 如果每个level都快到达了阈值, 一次压缩就导致多层都触发压缩, 加剧**写放大**
- Size-Tiered compaction
    * N个level, 每个level多个SSTable
    * 统一level中的SSTable可能存在交集, 扫描时需要多个SSTable, 导致**读放大** (读到重复数据)
    * 压缩: 同级的所有SSTable压缩, 但不与i+1层SSTable压缩, **减缓写放大**
- Tiered & Leveled compaction(混合模式)⭐
    * 层级低的level使用leveled模式, 层级高的leveled使用tiered模式
- FIFO compaction
    * 不分level, SSTable维护在一个FIFO链表中, 链尾会被淘汰
    * 简单但会丢数据, 适合时序数据库等时间敏感
- 墓碑: 因为row key可能存在多个SSTable中, 需要添加墓碑防止其他SSTable复活delete的row


#### OB中的合并

- 多种SSTable合并策略, 见机行事
- 全量合并
- 增量合并
- 渐进合并
- 轮转合并: 轮流做主, 轮流到后台合并
- 并行合并


### 存储格式

- 存储分层结构
- 内存存储格式
- 磁盘存储格式

OB是一个HTAP db, 支持行存和列存, 也因此引入了宏块(macro)和微(micro)块的概念。宏块是读的基本单位, 内含若干微块, 大小固定。微块可能是行存格式也可能是列存格式。宏块是内存布局如下: **尾部保存了微块的索引**

```
| Macro Block Header | Micro Block | ... | Micro Block Index |
```

- OB中的LST Tree
    * MemTable
    * Major SSTable, 最新版本信息的SSTable
    * Minor SSTable, 多个版本信息的SSTable


#### 主表查询

> 从新到旧的原则 + 火山模型: MemTable > Major SSTable > Minor SSTable


因此主表查询(Query)等于: MemTable + Major SSTable。当MemTable中有完整记录就不需要继续查询旧数据。

Scan的情况, 火山模型, 先提取出数据范围, 然后构造各个SSTable(table 集合)的迭代器依次迭代一行比对新旧然后提取。

需要比对多个SSTable于是引入了一个 **loser tree** 的抽象简化比较操作。它的任务是维护每个迭代器的当前行, 主键排序和从新到就维护主键相等的函数。


#### 索引表查询

索引表结构, 索引到主键的映射: `| index | primary key |`。查询过程就是先查索引表获得到主键, 然后用主键查主表。


#### 查询优化

- 预取到内存
- 过滤算子下压: 提前过滤

TODO: 什么是向量化

### 查询优化器

- 作用: 找到最优的执行计划
- 查询改写: 转换成更易优化的形式, 更小的表, 更多类型的链接算法
    * 基于规则改写: 如多表可以压缩
    * 基于代价改写: 多种方案可能都可行, 但是代价不确定是否更优
    * OB中会不断迭代基于规则改写和基于代价改写, 收敛后输入物理引擎
    * 详见社区相关[文章](https://www.oceanbase.com/docs/community-observer-cn-10000000000017056)等
- 查询优化: 整物理执行的过程, e.g. 谁左谁右做链接
    * 计划枚举: e.g. Left-deep tree, 枚举分配算子: 投影, 选择, 分组, join等
        + 枚举会消耗大量计划空间
        + ob引用一种skyline算法用于过滤, 提前找出最优解集合
        + 在表小于10的使用使用动态规划枚举
            + 是否回表, 是否interesting order, 是否可以抽取query range
    * 代价计算


### 执行引擎

- 简介
- 向量化执行
    * 每次迭代不再返回一行, 而是返回一batch。计算也是基于batch计算
    * 算子向量化: e.g. sum(), 排序后分哈希并行求
    * 存储向量化: 行列混存, 充分利用列存的特性
- 并行执行
    * 任务以DFO(data flow operator)为单位进行调度, 每个DFO会启用多个worker处理, 每次会调度起父子DFO
- 执行模式
    * Local: 数据在本地
    * Remote: 数据不在本地, 向远端发送SQL/Plan
    * DAS(Data Access Service)
    * 并行(PX)


### 事务引擎






