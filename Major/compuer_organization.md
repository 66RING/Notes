---
title: Computer Organization
date: 2019-12-10
tags: 
---

## 第二章

### 2.2 浮点数的表示

- 单精度IEEE754格式
    - 真值变二进制数
    - 二进制数移位变成$(-1)^S \times{2^e} \times {1.M}$
        - $E = e + 01111111$ 加上偏移量127得到E
    - 根据公式得到S,E,M后按照格式保存

### Representation of data

#### Overflow of Complement
If a complement calculations run out of scope, it overflow. So we need to be careful.


## Part.3

### Cpu Subsystem

Major component
- Temporal parts
- Control unit
    - logic controler
    - program controler
- Cache, to balance cpu(hight speed) and register(low speed)
- Register(heap)
    - general register
    - temporary register
    - IR(instruction register)
    - PC(process counter)
    - PSW(program status register)
    - MAR(address register)
    - MBR(binary register)
    - SP(stack pointer)
- Arithmetic unit

Control mode
- Synchronous
- Asynchronous

Instruction type
- CISC(Complex Instruction Set Computing)
- RISC(Reduced Instruction Set Computing)
- CISC-\>RISC



