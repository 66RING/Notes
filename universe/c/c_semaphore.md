---
title: 进程/线程间使用信号量通信
date: 2020-05-03
tags: 
- c
- semaphore
- IPC
mathjax: true
---

# 信号量通信

## semaphore.h

### 信号量创建和初始化sem\_init

```c
int sem_init(sem_t *sem, int pshared, unsigned int value);
```

- `sem`，初始化的信号量
- pshared参数：
    * pshared为0，表示信号量在线程间共享，放置在**所有线程可见**的位置，如全局变量
    * pshared非为0，表示信号量在进程间共享，放置在**所有进程可见**的位置，如使用`sem_open`、`shm_open`、`mmap`等访问共享内存
        + 因为通过`fork()`创建的子进程会继承父进程的虚拟地址，但是cow会让其有独立不同于父进程的物理空间，导致地址相同但值不同
- `value`信号量初始值
- 成功返回0，失败返回1

- `int sem_post(sem_t *sem);`
	* 信号量值加1
- `int sem_wait(sem_t *sem);`
	* 信号量值减1
- `int sem_destroy(sem_t *sem);`
	* 信号量销毁

**非亲缘关系使用**：`sem_open`通过信号量名打开

```c
sem_t *sem_open (const char *__name, int __oflag, ...)
```

- `__oflag`设置打开方式，`fcntl.h`中定义了很多常用方式
	* `O_CREAT`，如果不存在，则创建
	* `O_RDONLY`，只读模式 
	* `O_WRONLY`，只写模式 
	* `O_RDWR`，读写模式
	* `O_APPEND`，末尾追加
	* `O_EXCL`，如果已存在，则返回 -1，并且修改errno的值
	* `O_TRUNC`，如果文件存在，并且以只写/读写方式打开，则清空文件全部内容 
	* `O_NOCTTY`，如果路径名指向终端设备，不要把这个设备用作控制终端。
	* `O_NONBLOCK`，如果路径名指向FIFO/块文件/字符文件，则把文件的打开和后继I/O设置为非阻塞模式
	* `O_DSYNC`，等待物理 I/O 结束后再 write。在不影响读取新写入的数据的前提下，不等待文件属性更新。 
	* `O_RSYNC`，read 等待所有写入同一区域的写操作完成后再进行
	* `O_SYNC`，等待物理 I/O 结束后再 write，包括更新文件属性的 I/O


## sys/sem.h

另一种使用信号量的方式

```c
int semget(key_t key, int nsems, int semflg);
```

成功返回信号量标识符，否则返回-1

- `key`，表示该**信号量集**的整数
- `nsems`，创建的信号量的数目
- `semflg`，标记位，设置权限等，如`IPC_PRIVATE`表示进程私有

```c
int semctl(int semid, int semnum, int cmd, ...);
```

操作和控制信号量

- `semid`，semget返回的信号量标识符
- `semnum`，所操作信号在信号量集中的编号
- `cmd`，操作方法
	* `SETVAL`设置信号量值
- 最后参数与cmd参数有关，传入如下共用体执行具体操作

```c
union semun{
    int val;                        //cmd为SETVAL时，用于指定信号量值
    struct semid_ds *buf;            //cmd为IPC_STAT时或IPC_SET时生效
    unsigned short *array;            //cmd为GETALL或SETALL时生效
    struct seminfo *_buf;            //cmd为IPC_INFO时生效
};
```

```c
int semop(int semid, struct sembuf *sops, unsigned nsops);
```

使用信号量，对信号量的值操作

- `semid`，`semget`返回的信号量标识符
- `spos`，操作方法数组，对信号量集的信号量操作集合
- `nsops`，sops数组的长度

sembuf结构如下：

```c
struct sembuf{
  short sem_num;     //信号量在信号量集中的编号
  short sem_op;      //操作方法，-1表示P操作，1表示V操作
  short sem_flag;    //标志位，通常SEM_UNDO让进程退出自动释放信号量
};
```






