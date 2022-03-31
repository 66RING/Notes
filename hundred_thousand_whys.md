---
title: Hundred Thousand Whys
date: 2020-9-17
tags: 
- tips
- tools
mathjax: true
---

# stack

## 为什么访问外存慢

一般都会认为访问外存慢是因为IO慢，要提速添加内存中的缓存就可以了就像redis一样。但是这其实是不正确的，**因为操作系统本身就提供外存到内存的缓存服务了呀**。**真正慢的原因是: 外存数据结构和内存数据结构的转码。**试想一下，在内存中，你的数据结构可以直接使用各种指针访问内存，那这些指针在外存中该怎么办？有些同学就会说啦序列化然后复制粘贴到外存就可以了呀。但是要知道内存是归操作系统管理的，你直接序列化复制粘贴到外存，那外存复制粘贴到内存一定还可以吗？内存一定空闲吗？等等原因，我们还要额外考虑**外存数据结构**。

所以redis之类，提升了系统速度的原先不是因为它是利用了内存和外存的速度差，而是因为它是"内存中的数据库"，不需要内外存数据结构转码

## inc文件

inc文件完整名称应该是"include"是专门用于`#include`的。因为一般来说，声明放在".h"文件，定义放在".c"，而".inc"文件一般同时包含声明和定义，因为要一"#include"就能使用。

使用inc文件将一些代码打包，使用的时候再`#include`可以提高开发效率。

下面的需要讨论的内容：

Q1: 那为何不直接使用".h"+".c"的方式? 我觉得也是可以的，不过通过观察qemu项目结构可以发现，qemu项目中需对函数都是`static`标注的，也就是说仅可以在当前文件使用。要想使用这些函数可以通过引入`.inc`文件的方式实现。

Q2: Q1中提到函数使用`static`标注，那为什么要`static`标注？我觉得是为了编译系统的良好组织和解决"重新定义"的问题。1. 因为在大型项目中可能会需要用到很多工具函数，这些工具函数甚至虚拟机相关的函数可能会重名。2. qemu的编译系统是可以选择哪些文件需要加入编译的，如果".inc"拆分成很多的".c"和".h"会让编译系统的变得比较零散。


## RISCV hart

> HARdware Thread

相当于独立的处理器，在软件看来hart是一种资源，一个hart就是一个执行执行流。类比x86超线程


## RISCV命名规则

`RV[xxx][abc-xyz]`

- `[xxx]`多少位机
- `[abc-xyx]`实现的模块集合
- eg. `RV64IAM`

## 目标三元组

CPU 架构、CPU 厂商、操作系统和运行时库, eg: `riscv64gc-unknown-none-el`, CPU 架构是 riscv64gc，厂商是 unknown，操作系统是 none，elf 表示没有标准的运行时库（表明没有任何系统调用的封装支持），但可以生成 ELF 格式的执行程序。


## Why rust say it want to replace c++

Rust has a specfial  *Memory Management* , which is an **COMPILE-TIME memnory management base on Ownership, Borrowing and Lifetime** .

- C family: manually manage pointers and  **trust programmer access them and free correctly**. Otherwise wil cause run-time errors.
- Java: Run-time GC
- Rust: Ideally not to manage pointers manually, it has a compile-time borrow check.


## Diff in DOS, Mac and Unix file

The old Teletype machine used two character to satrt a new line. One to move the carriage(return \<CR\>) back to first position.

Back in the day, storage was expensive. Some people decided they did not need two character for end-of-line.

- The Unix people decided they use `Line Feed` only for end-of-line
- The Apple people decided they use `<CR>`
- The Window forks decided to keep the old `<CR><LF>`


## Why we need to 1==i

We need to write `1==i` but not `i==i` in conditional statements.

If we accidentally make a mistake `1=i`, compile with not pass


## 什么是websocket

传统的http的客户端服务端模式是：客户端请求，等待服务端响应。因此服务端的相应很依赖于客户端的请求

socket的客户端服务端模式，一般通过tcp长连接，就可以实现服务端的主动相应。这在需要立即作出状态变更的场景是非常必要的，如网络聊天室、股票信息等。

传统的http一般通过轮循的方式来检测状态的变更，非常浪费带宽。因此，我们需要http，也需要socket，由此websocket就诞生了。


## 什么是REST接口规范

REST规范的http接口如下：

- `GET`，用于**查**操作
- `POST`，用于**增**操作
- `PUT`，用于**改**操作
- `DELETE`，用于**删**操作


## 何时发生用户态和内核态的切换？

- 用户态到内核态
    * 申请外部资源时
        + 硬件
        + 系统调用
        + 中断
        + 异常
- 内核态到用户态
    * 需要在内核态处理的事完成时


## 期望效用

圣彼得堡悖论: 抛一枚硬币，直到出现正面为止。第n次出现正面将获得$2^{n-1}$元。虽然按照期望，一直玩下去收益是无穷大，但是有人愿意投入那么多钱玩吗。此说明不能单纯靠期望来做决策。

同样的财富对于不同阶级来说意义不同，因此决策应该和原有财富有很大关系，因此在期望理论之后又提出了效用理论。

用对数描述效用：

$$U(x) = lnx - lnA$$

其中x表示收益，A表示原有财富。效用期望就是效用乘以相应的概率。

但是如果将奖金调整为$e^{2^{n-1}}$，则又会碰到同样的问题。因此理论上可以通过调整新的金额来制造新的悖论。


## 库克之于苹果

看似库克当了苹果CEO后就一直抱怨苹果没有了创新。但也可能是乔布斯的功绩太过辉煌。

那么如果乔布斯继续当然CEO苹果会做得比现在好吗。库克其实做了很多乔布斯不会做的，比如环保，社会服务，慈善等。也许也因为这些品牌担当才会有现在苹果2万亿美元的市值也说不定。


## 自我介绍黄金公式

人的注意力像条U型曲线，一两分钟因此要要先将成绩好的

- 开头，展示价值
- 中段，主动引导
    * PST结构，P从事的职务，S成功的经验，T转职的原因
- 结尾，设定对话方向


## 为何理财

**为了战胜通货膨胀**

如6年前最贵最高配手机，与现在最贵最高配手机比较，价格翻了3倍。你的钱越来越没有价值，如果放银行，将会被通货膨胀吞噬，6年后缩小3倍。


## linux系统常用目录

- /bin
	* binary, 所有用户都可使用的程序
- /boot
	* 启动相关
- /dev
	* device, linux下"一切皆文件"，可以以文件的形式读写设备
- /etc
	* etcetera(等等), 一般管理配置文件
- /home
	* 用户主目录，每个用户一个文件夹
- /lib
	* 保存系统基本的动态链接库
- /lost+found
	* 非法关机时里面会出现一些文件
- /opt
	* optional, 额外软件保存的目录
- /proc
	* process, 进程当成文件，可以通过文件管理进程
- /root
	* 超级管理员目录
- /run
- /sbin
	* Super user Binary，超级管理员使用的程序
- /srv
	* 服务启动后需要提取的数据
- /sys
	* sysfs文件系统，系统信息以文件形式读写
- /tmp
	* 临时文件
- /usr
	* unix shared resource，共享资源
	* /bin -> /usr/bin
	* /sbin -> /usr/sbin
	* /usr/src, linux源码
- /var
	* variable, 不断扩大的东西，如日志文件


## Markdown syntax that beautiful enough

Headings

```
标题紧跟
任意数量的'='表示H1
任意数量的'-'表示H2

H1
==============================

H2
------------------
```

Ordered Lists

```
数字加句点表示有序列表，必须数字1开始，但后续不必再有序

1. first
2. second
3. third
4. fourth
```

显示\`

```
转意`` ` ``，两个\`包裹的内容将会被保留
```

直接显示url或邮箱

```
<https://www.markdownguide.org>
<fake@example.com>
```

**格式化引用**

```
Reference-style links包含两部分

1. 句子中的引入，可以简写成[link]
[label][link]

2. 可以写在文中任意位置，可以有如下格式
[link]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle
[link]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle "Hobbit lifestyles"
[link]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle 'Hobbit lifestyles'
[link]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle (Hobbit lifestyles)
[link]: <https://en.wikipedia.org/wiki/Hobbit#Lifestyle> "Hobbit lifestyles"
[link]: <https://en.wikipedia.org/wiki/Hobbit#Lifestyle> 'Hobbit lifestyles'
[link]: <https://en.wikipedia.org/wiki/Hobbit#Lifestyle> (Hobbit lifestyles)
```
