---
title: 读写锁与RCU
author: 66RING
date: 2022-05-19
tags: 
- os
mathjax: true
---


# 读写锁实现

本质上仍是通过**写锁来互斥**，只是读锁减小的锁影响的范围，读锁仅用于保证`reader++`的安全。

可以看到虽然看起来读者"不互斥"，但是内部读者还是存在锁竞争的。所以引入RCU

```c
struct rwlock {
	int reader;
	struct lock read_lock;
	struct lock write_lock;
};

void rlock(struct rwlock *lock) {
	lock(&lock->read_lock);
	lock->reader += 1;
	if(lock->reader == 1) 		// 第一个Reader
		lock(&lock->write_lock);
	unlock(&lock->read_lock);
}
void unrlock(struct rwlock *lock) {
	lock(&lock->read_lock);
	lock->reader -= 1;
	if(lock->reader == 0) 		// 没有Reader了
		unlock(&lock->write_lock);
	unlock(&lock->read_lock);
}
void wlock(struct rwlock *lock) {
	lock(&lock->write_lock);
}
void wunlock(struct rwlock *lock) {
	unlock(&lock->write_lock);
}
```

- 读者也要上读者锁
- 关键路径额外开销
- 易用


# RCU(Read Copy Update)

基于原子指令，但是很多原子指令对要操作的对象的大小有限制，所以思路就是：**原子指令修改指针**

1. 新增：插入链表，原子指令该指针指向
2. **修改**: 先拷贝一份原对象的副本，在副本修改完成后，原子指令修改指针执行新副本

- **局限**
	1. 每次修改都有额外的**拷贝开销**
	2. **旧拷贝何时释放** 因为虽然修改了指针，但是旧拷贝可能仍有人在使用

引入 **RCU宽限期** 为了正确释放旧拷贝。因为我们知道发生RCU的时刻，和访问旧拷贝的开始和结束。所以我们可以根据时间，在最后一个对旧拷贝的访问结束时释放就旧拷贝。

所以，RCU对计时器有求比较高。**必须要计时器吗?加一个引用计数行不行?** 引用计数的不可以的，因为在并发访问情况情况下，为了保证引用计数的正确性就需要互斥，这样就有回到了锁竞争的效率问题，就忘了RCU的初衷。


- 无锁
- 写者开销大
- 较繁琐

