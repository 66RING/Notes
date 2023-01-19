---
title: AddressSanitizer(ASan)使用
author: 66RING
date: 2000-01-01
tags: 
- tools
mathjax: true
---

# AddressSanitizer

memory debuging in c.

## 编译

编译

```
# 需要安装address sanitizer库
gcc <filename> -g3 -fsanitize=address
```

cmake中的编译(把flag加上)

```
set(CMAKE_C_FLAGS,"${CMAKE_C_FLAGS} -fsanitize=address -g3")
set(CMAKE_EXE_LINKER,"${CMAKE_EXE_LINKER} -fsanitize=address -g3")
```


## 错误信息查看

测试用例: 下面这个程序存在安全隐患, 我们将4byte长的数据拷贝到了容量为3的容器中。但仍能编译运行并输出预测的结果。

```c
#include<stdio.h>
#include<string.h>

int main() {
	char str1[] = "1234";
	char str2[3];
	strcpy(str2, str1);
	printf("%s", str2);
	return 0;
}
```

启用address sanitizer编译:

```
$ gcc <filename> -g3 -f sanitize=address -o a
$ ./a
=================================================================
==800==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fffc2c19013 at pc 0x7f89a4a12e92 bp
0x7fffc2c18fd0 sp 0x7fffc2c18778
WRITE of size 5 at 0x7fffc2c19013 thread T0
    #0 0x7f89a4a12e91 in __interceptor_strcpy /usr/src/debug/gcc/libsanitizer/asan/asan_interceptors.cpp:425
    #1 0x55ad811222d1 in main /home/ring/Documents/code/c/tt/a.c:7
    #2 0x7f89a47ef28f  (/usr/lib/libc.so.6+0x2328f)
    #3 0x7f89a47ef349 in __libc_start_main (/usr/lib/libc.so.6+0x23349)
    #4 0x55ad811220e4 in _start ../sysdeps/x86_64/start.S:115

<============ 调用栈信息

Address 0x7fffc2c19013 is located in stack of thread T0 at offset 51 in frame
    #0 0x55ad811221c8 in main /home/ring/Documents/code/c/tt/a.c:4

  This frame has 2 object(s):
    [48, 51) 'str2' (line 6) <== Memory access at offset 51 overflows this variable
    [64, 69) 'str1' (line 5)
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext
or vfork
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow /usr/src/debug/gcc/libsanitizer/asan/asan_interceptors.cpp:425 in __interceptor_strcpy
Shadow bytes around the buggy address:
  0x10007857b1b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b1c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b1d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b1e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b1f0: 00 00 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1
=>0x10007857b200: f1 f1[03]f2 05 f3 f3 f3 00 00 00 00 00 00 00 00
  0x10007857b210: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b220: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b230: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b240: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10007857b250: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

<============ 内存访问信息

Shadow byte legend (one shadow byte represents 8 application bytes):  <===== 每个00这样的数表示8个byte
  Addressable:           00                                           <===== 其中00表示可访问
  Partially addressable: 01 02 03 04 05 06 07                         <===== 数字表示几个可访问
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==800==ABORTING
```

## 大项目保留现场(gdb coredump)

你的项目crash了, 返回了上面asan的信息, 但你还是很难发现有用的信息，比如asan只打印了调用栈。所以你可以开启coredump, 然后转用gdb打开查看现场。

这时你就可以看看调用栈哪里出了问题了

- gdb tips
    * 查看各个调用栈: `(gdb) frame [n]`
    * 查看本地变量: `info locals`

TODO:




