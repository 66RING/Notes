---
title: C语言中使用管道进行进程间通信
date: 2020-05-04
tags: 
- c
- pipe
mathjax: true
---

## pipe

```c
int pipe(int pipefd[2]);
```

创建一个管道，返回其文件描述符fd。`pipefd[0]`用于读，`pipefd[1]`用于写。

由于其是基于文件描述符(fd)的管道，所以仅能在 **有亲缘关系的进程** 间共享，也就是说要通过管道做进程间通信仅能在其父子兄弟进程间通信。

因为每个进程都维护自己的**文件描述符表** fd table，fork时子进程继承了父进程的fd table。所以父进程已存在的fd table将会继承到子进程，fd指向的文件同父进程。即两张fd table上，部分内容相同。

若子进程继续打开文件将是在自己的table上加内容，不会影响父进程。父进程继续打开的文件也不会影响子进程，因此无亲缘关系的进程fd table一开始就没有相同的部分，也就操作不到同一个管道。

可通过读写函数操作文件描述符：

```
write(fd, buf, sizeof(buf)
read(fd, buf, sizeof(buf)
```


## fifo

```c
int mkfifo(const char *pathname, mode_t mode);
```

`mkfifo`将创建一个fifo文件，其中`pathname`表示要创建的fifo文件，`mode`表示读写权限如`0664`。如fifo名字所示，其内容是同队列先进先出的。可以用来作为 **无亲缘关系的进程的管道** 

这种管道是基于fifo文件的，所以只需要知道文件的位置，就能在进程间使用。

`mkfifo`创建好文件后可以通过`open()`函数打开文件进行读写。就像pipe操作一样，`open()`函数原形如下

```c
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h> 

int open( const char * pathname, int oflags);
int open( const char * pathname,int oflags, mode_t mode);
```

`open`打开一个文件，**返回文件描述符**。常用的`oflags`有，可以使用`|`拼接

- `O_RDONLY`，只读模式 
- `O_WRONLY`，只写模式 
- `O_RDWR`，读写模式
- `O_APPEND`，末尾追加
- `O_CREAT`，如果不存在，则创建
- `O_EXCL`，如果已存在，则返回 -1，并且修改errno的值
- `O_TRUNC`，如果文件存在，并且以只写/读写方式打开，则清空文件全部内容 
- `O_NOCTTY`，如果路径名指向终端设备，不要把这个设备用作控制终端。
- `O_NONBLOCK`，如果路径名指向FIFO/块文件/字符文件，则把文件的打开和后继I/O设置为非阻塞模式
- `O_DSYNC`，等待物理 I/O 结束后再 write。在不影响读取新写入的数据的前提下，不等待文件属性更新。 
- `O_RSYNC`，read 等待所有写入同一区域的写操作完成后再进行
- `O_SYNC`，等待物理 I/O 结束后再 write，包括更新文件属性的 I/O

值得注意的是open打开只读的fifo文件时会**阻塞**，直到有别的进程用写模式打开该文件，同理打开只写文件也会阻塞。可以通过`O_NONBLOCK`标志来避免阻塞，但就不能同步了

`open`根据文件路径返回fd后就可通过读写函数操作文件描述符：

```
write(fd, buf, sizeof(buf)
read(fd, buf, sizeof(buf)
```


