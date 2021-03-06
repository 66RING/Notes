---
title: 电路
date: 2020-6-7
tags: 
- 电子管
- 电路
---


## 半导体的工作原理

半导体的材料一般是电子不多不少(不易获得也不易失去)的，如:硅(4最外层4电子)

电子多，说明核带正电多，对电子吸引力强; 电子少，说明核带正电少，对电子吸引力弱。


### 晶体二极管

纯硅电子不易得也不易失，我们称之为 **本征半导体** 。如果我们在硅中加入一点磷(最外层5电子)，则将不是在最外出4电子的基础上多了一个电子。这时这个电子将相对"自由"。这个整体带的电子比稳定的4电子多，我们就叫它N型半导体(Negetive)。

如果我们在本征半导体参入堋(最外层3个电子)，整体将变得渴望1个电子，我们将这个空的地方(为到达4电子)称为空穴。这些空穴有渴望电子的能力。相对4电子结构，空穴是Positive的，因此称为P型半导体。

采用特殊的技术，把P型半导体和N型半导体(P、N整体都是中性，别被PN误导)拼接在一起。那么由于P型半导体的空穴浓度高，N型半导体的空穴浓度低，空穴就会从P扩散到N;同理N型半导体的电子扩散到P型半导体，就像液体中的扩散一样。但扩散不会一直发生，当N中的空穴变多，变得带正电，P中的电子变多，变得带负电，形成电场。

随着电场的形成，电场将试图将P的电子拉到N，随着P的电子减少，电场减弱，扩散又发生，最后扩散和电场达到动态平衡。

- 如果我们向N区通正电，新增的电场与内部的电场方向一致，电场将会变强。虽然仍会处于动态平衡，但是电场抑制了粒子移动，从而阻断电流。
- 如果我们向P区通整点，N区通负电，将压缩内电场，电子的束缚将变小从而导通电流
- 于是晶体二极管就诞生了


### 晶体三极管

晶体三极管结构如下：

| 外       | N | P | N | 内 |
|----------|---|---|---|----|
| 通电电极 | + | + | - |    |
| 位置     | 3 | 2 | 1 |    |

每个PN接触面会形成PN节，一般0.7V的电压会导通PN节。

如果1号位和2号位电压大于0.7V，就通电了。需要注意的是，设计时，1号位故意参杂很多5电子元素，使得电子浓度很高;而2号位的半导体很薄，很难一次消耗掉这些涌入的电子;当2号涌入了很多电子又无法消耗，那么就打破了2号和3号的动态平衡。而且3号设计得很大电子浓度低，2号扩散过来的电子很快会被3号收集。又因为3号通的正电，电子得到了一个快速的泄洪通道，迅速通过电源正极。

这样一来，我们可以认为2号和3号通电了。又因为2号电子来自1号，所以1号也和3号通电了。

2号极小的信号改变就会导致1号电子涌入的巨大变化，从而引起1号与3号之间电流的巨大变化。

- 1号连接负极，称为发射极
- 2号相当于阀门，操作这原始信号，称为基极
- 3号收集电子，称为集电极


### 场效应管

将两个N型半导体浸如一个大的P型半导体中，两个N型半导体分别接入正极和负极，P型半导体接入正极，但与正极间隔这一个电容。

|          |   | N | P     | N |   |
|----------|---|---|-------|---|---|
| 通电电极 | P | - | +电容 | + | P |
| 位置     | P | 1 | 2     | 3 | P |
|          | P | P | P     | P | P |

- 1号由电源不断提供电子
- 2号不断吸引电子，但不会快速消耗电子
- 3号快速消耗电子

由于2号电子处聚集了大量电子，3号消耗了很多电子，于是P型半导体由于电子聚集的位置与3号N型半导体由于电子消耗，半导体的类型发生了改变。即2号与3号之间，2号成了N型，3号成了P型，电子将扩散到3号并被快速消耗。又由于1号源源不断提供电子，不断涌向2号。于是1号和3号在2号的控制下形成了通电回路。

- 1号提供电子，称为源极
- 2号像栅栏一样控制电路导通，称为栅极
    - 栅极的正负控制这电路的通阻，如果用01表示同断，那么计算机科学就开始了
- 3号称为漏极






