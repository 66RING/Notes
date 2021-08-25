---
title: 还没想好TODO
date: 2020-08-04
tags: 
- OS
mathjax: true
---

# 奇技淫巧

有趣的书：  [*Hacker's Delight*](https://www.hackersdelight.org/) 

- gcc -E仅预编译，再脚本处理以下，格式化一些就能得到宏展开的代码
- gdb -> r -> SIGSEGV


## bitset

### bitsize

统计1的bit数。先看长度2bits的情况。

```
B1: a b
B2: c d

B1处理：
		  B1 &  1    0 b
  + (B1 >> 1) & 1    0 a
  -------------------------------
              x x    x x

结果的2bit数就是1的个数。同理B2
```

长度4bit的情况就相当于两个2bit，再将两个2bit数1个数结果相加。以此类推更多位数。

```
int bitset_size(uint32_t S){
	S = (S & 0x55555555) + ((S >> 1) & 0x55555555);
	S = (S & 0x33333333) + ((S >> 1) & 0x33333333);
	S = (S & 0x0F0F0F0F) + ((S >> 1) & 0x0F0F0F0F);
	S = (S & 0x00FF00FF) + ((S >> 1) & 0x00FF00FF);
	S = (S & 0x0000FFFF) + ((S >> 1) & 0x0000FFFF);
	return S;
}
```

这个技巧在嵌入式机器中可能表现比较好，但在现代处理器中未必，因为其存在很强的数据依赖问题。


### lowbit

`lowbit(x)`仅保留最右边的1。这是一个思路的问题，我们希望实现如下变化：

```
x: +++++100  ->  00000100
```

可以尝试变换出形式类似的如:

```
x-1: +++++011
~x : -----011

((x-1) & ~x) + 1 = lowbit(x)
```

### log2

log2相当于找最高位(最左边)的1。可以采用分治的办法处理。


# 并发控制

看似无害的数据竞争也会产生危险，因为编译器的优化。如读竞争：

```
for 4 thread each: Atom(done++)

while(done!=4){}
```

done写通过原子指令解决了数据竞争，但是done的读没有处理数据竞争问题。经编译器优化后可能`mov %eax, done`将done赋值给eax寄存器，再用eax寄存判断，而eax寄存器又是线程本地的，所以如果eax赋值前done为4则正确，但是可能不为4。


# 虚拟化


操作系统为程序提供虚拟

## fork

fork -> 一次又一次的虚拟化

```c
// hello.c
for(int i=0;i<2;i++){
	fork();
	printf("Hello world\n");
}

wait....
```

画出以上状态机，打印了几个Hello？6

```
$ hello | wc -l
```

又打印了几个Hello？8。因为printf不是直接输出到stdout，而是先输出到libc的一个缓冲区。


### fork boom

```sh
$ :(){ :|:& }; :
```

PS: shell中允许`:`作为函数名。链式反应





