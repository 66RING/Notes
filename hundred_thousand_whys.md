---
title: Hundred Thousand Whys
date: 2020-9-17
tags: 
- tips
- tools
mathjax: true
---

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



