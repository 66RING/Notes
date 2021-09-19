---
title: cache一致性协议：MESI
date: 2021-02-07
tags: 
- architecture
- cache
mathjax: true
---

# Preface

MESI stand for

- Modified
	* cpu的cache中的内容与主存不一致
- Exclusive
	* 只有一个cpu的cache中有主存数据的副本
- Shared
- Invalid

- 处于共享态(Shared)的cpu修改cache的同时也要修改主存，即写直达法(write through)
- 处于独占态(Exclusive)或修改态(Modified)的cpu修改cache的时，采用写回法(wirte-back)：只修改cache不修改主存，当cache被替换后才写回主存同步
- cache间通信使用SHARED bus line，来传递cache一致性相关的消息


TODO

- cpu会监听其他cpu的读写操作
	* 如果读的与其他cpu中cache中TODO一样，则通过SHARED bus通知其他cpu，置共享态(Shared)，其他cpu转而从cpu的cache取
	* cache数据修改后也是Exclusive，修改操作会被监听到，置为Invalid
	* Exclusive再修改就是Modified
	* 别的cpu读，被Modified的cpu监听到，cpu写回，通过share bus改share，从cache取

# 实现

## 理解

**Shared bus line** 专门用于"Shared"相关的

- Shared态的cpu，必须??写直达，且改变为Modified
	* 否则Shared态cpu间数据都不一样了还怎么称得上Shared
- Exclusive和Modified都采用write back



## 状态机

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/Major/architecture/MESI/MESI.png" alt="">

- Shared
	* TODO: sum
	* 自己读：
	* 自己写：
	* 其他cpu写：
	* 其他cpu读：
- Invalid
	* TODO: sum
	* 自己读：
	* 自己写：
	* 其他cpu写：
	* 其他cpu读：
- Modified
	* TODO: sum, 不一致..
	* 自己读写：状态不变，write back地写
	* 其他cpu读：读的cpu从Modified的cpu读，cpu们变为Shared
	* 其他cpu写：变为Invalid
- Exclusive
	* TODO: sum，一致、有cache...
	* 自己读：状态不变
	* 自己写：变为Modified，write back地写
	* 其他cpu写：变为Invalid
	* 其他cpu读：变为Shared

**状态迁移图 -> 时序逻辑电路**


