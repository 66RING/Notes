---
title: AT&T与Intel汇编语言的比较
date: 2020-6-10
tags: AT&T, Intel, 汇编语言
---


GCC采用的是AT&T的汇编格式，也叫GAS(Gnu Assembler)格式;微软采用Intel的汇编格式


## 寄存器命名

- ATT的汇编格式中，寄存器名前要加上"%"前缀
- Inter格式中不用


## 操作数的顺序

- ATT目标操作数在源操作数的右边
- Intel目标操作数在源操作数的左边
- 正好相反

| AT&T            | Intel       |
|-----------------|-------------|
| movl %eax, %abx | mov ebx,eax |


## 常数/立即数的格式

- ATT的汇编格式中，立即数要加上"$"前缀
- Inter格式中不用


## 操作符长度标识

- ATT汇编格式中操作符的后缀"b", "w", "l"分别表示操作数为字节(byte, 8位)、字(word, 16位)、长字(long, 32位)
- Intel中操作数长度会根据寄存器长度而定，也可用"byte ptr"、"word ptr"等前缀来表示


## 寻址方式

| AT&T                                       | Intel                                |
|--------------------------------------------|--------------------------------------|
| imm32(basepointer,indexpointer,indexscale) | [base+indexpointer*indexscale+imm32] |

两种寻址效果都是：`imm32 + basepointer + indexpointer*indexscale`


## 跳转指令

- ATT汇编格式中，绝对跳转和调用指令(jump/call)的操作数前要加上"\*"作为前缀
    - 远程跳转指令和远程子程序调用指令的操作码在ATT中为"ljump"和"lcall"
    - 对应的返回指令为`lret`
- Intel中不用
    - 远程跳转指令和远程子程序调用指令的操作码在Intel中为"jmp far"和"call far"
    - 对应的返回指令为`ret`

- ATT的汇编格式中，跳转指令有点特殊
    - 直接跳转，即跳转目标作为指令的一部分编码，如`ljump $a`
    - 间接跳转，即目标从寄存器或储存器位置中读出的。写法在"\*"后面跟一个操作数指示符
        - `ljump *%eax`用寄存器%eax中的值作为目标
        - `ljump *(%eax)`用寄存器%eax中的值作为读入地址，从存储器中读出跳转目标






