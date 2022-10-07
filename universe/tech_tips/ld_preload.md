---
title: LD_PRELOAD劫持
author: 66RING
date: 2022-10-06
tags: 
- tech
mathjax: true
---

# LD_PRELOAD tips

很多程序都需要链接共享库才能运行, 链接共享库默认会从系统`/usr/lib`目录中查找, 然后链接`.so`文件。而且对于同名函数, 只会识别最先出现的那一个。指定`LD_PRELOAD`变量就能让链接器先去找`LD_PRELOAD`指定的路径再去找默认路径, 从而实现对函数劫持。

详见`man ld.so`

## 实验例子

我们可以通过`strace`查看一个程序执行过程中使用到的系统调用。再配合上`man 2`就可以查看系统调用的详细信息, 以便后续覆盖。

来一个经典的篡改`whoami`

使用`strace`查看`whoami`的调用。可以看到调用了`geteuid`, 使用`man 2 geteuid`查看详情来做点伪造`uid_t geteuid(void)`可见返回的是一个用户id。

```
$ strace whoami
execve("/usr/bin/whoami", ["whoami"], 0x7ffe49e38f60 /* 102 vars */) = 0
brk(NULL)                               = 0x55b44d693000
...
geteuid()                               = 1000
...
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

使用`cat /etc/passwd`看看都有哪些uid

```
$ cat /etc/passwd
root:x:0:0::/root:/bin/zsh
```

那么可以简单返回个int, 然后`gcc --shared -fPIC a.c -o a.so`编译`.so`动态库

```c
// a.c
int geteuid() {
	return 0;
}
```

在`whoami`程序前使用`LD_PRELOAD`环境变量`LD_PRELOAD=$(pwd)/a.so whoami`, 可以看到返回`root`, 而我们现在并非root用户。

进一步的， 可以使用`ldd`命令查看运行都要链接哪些库。可以看到差异

```
$ ldd $(which whoami)
        linux-vdso.so.1 (0x00007ffdf99ee000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f3239db6000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f3239fcf000)

$ LD_PRELOAD=$(pwd)/a.so ldd $(which whoami) 
        linux-vdso.so.1 (0x00007ffd36bf8000)
        /home/ring/a.so (0x00007fc4ac3ff000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007fc4ac1f2000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007fc4ac410000)
```



## 伪造后怎么再去调用原来的?

使用`<dlfcn.h>`中的`dlsym(RTLD_NEXT, "name")`。`RTLD_NEXT`会找下一个出现的，返回函数指针, 从而实现对原始函数的调用。

使用`dlcn.h`需要引用dl库`-ldl`。`gcc --shared -fPIC -ldl a.c -o a.so`

```c
#include<stdlib.h>
#include<dlfcn.h>
int geteuid() {
	int (*f)(void); 
	f = dlsym(RTLD_NEXT, "geteuid");
	return f();
}
```

