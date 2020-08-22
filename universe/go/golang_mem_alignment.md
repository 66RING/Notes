---
title: Golang内存对齐
date: 2020-8-17
tags: golang, go
---


## <center>Golang内存对齐<center>

Cpu要想读取数据，需要通过 **地址总线** 把地址传输给内存。内存准备好数据输出到 **数据总线** ，交给CPU。

每根地址总线能表示一位，8根地址总线就能表示8位二进制数，即256个地址。因为表示不了更大的地址，所以就用不了更大的内存。

每次操作一字节太慢，那就加宽数据总线，要想一次操作一字节就至少需要32位数据总线。8字节就64位。这里每次操作的字节数就是所谓的  **机器字长** 。

从内存读取数据时通过"并行"操作同时从储存单元(DRAM芯片)读出数据。

``` 
----+-------+-------+-------+---- address
    |       |       |       |
    |       |       |       |
  chip1    chip2   chip3    chip4
    |       |       |       |
    0       0       1       1
----
0011...
假设有8个chip
```

每个储存单元共用同一个地址，各自从地址取出一位组成我们逻辑上认为的连续4字节数据。提高了内存访问效率。

采用着用设计，那么`address`就必须的8的倍数。如果非要错开一个格，那么错开后多出来的格子和其他不同，不能在一次操作中被同一个地址选中，所以这样的地址是不能用的。

CPU对这种情况的处理：多次读取。如取第一次读取的后7字节和第二次读取的前1字节拼接得到想要的8字节数据。但这样就影响了性能。

为了保证程序顺利高效的运行，编译器会把各种类型的数据安排到合适的地址并占用合适的长度，这就叫做 **内存对齐**。每种类型对齐值就是它的 **对齐边界**。

```
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|
用0~1存储int16的数据，对齐值(对齐边界)就是2
用4~7存储int32的数据，对齐边界就是4
```

内存对齐要求数据存储地址以及占用的字节数都是它对齐边界的倍数。因此，在上面的情况中，如果int16从0开始(0%2=0;2%2=0)，那么int32就要错开两字节，从4开始(4%4=0;4%4=0)。


### 如何确定每种类型的对齐边界

这与平台有关，go支持很多平台386、arm、mips等。常见的32位平台指针宽度和寄存器宽度都是4字节，64位平台上都是8字节。被Go语言称为寄存器宽度的这个值就可以理解为机器字长，也是平台对应的最大对齐边界。

数据对齐边界取数据类型大小和平台最大对齐边界中较小的一个。如果只根据平台最大对齐边界对齐，则容易浪费很多字节(跳过很多空闲的字节)。


### 结构体的对齐边界

取结构体中数据类型大小最大的成员，这个就是这个结构体的对齐边界。

- 1. 存储这个结构体的起始地址是对齐边界的倍数
- 2. 结构体每个成员存储时，都把起始地址当作地址0，然后再用相对地址决定自己放在哪
- 3. 结构体整体占用字节数需要是类型对齐边界的倍数，不够的话往后扩张

例子：

``` 
type T struct{
    a int8   // 1byte
    b int64  // 8byte
    c int32  // 4byte
    d int16  // 2byte
}

最大是8，所以改结构体对齐边界是8

第一个成员a从0开始;第二个成员要对齐，只能从8开始;第三个成员从16开始，占用16~19;第四个成员从20开始占用20~21。然后还需要向后扩张到23
```

思考：调整结构体成员的顺序是否能够优化内存？


