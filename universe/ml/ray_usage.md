---
title: Ray Usage
author: 66RING
date: 2025-08-30
tags: 
- 分布式
mathjax: true
---

# Ray Usage

Ray能方便使用多进程和分布式, 无需手动管理socket、线程池、消息队列等, 只需在函数或类前加上`@ray.remote`装饰器, 就能将其变成分布式任务或分布式对象。底层用了KV数据库自动管理这里metadata。

核心抽象

- task: function call
- actor: 分布式对象, AKA object的集合
- object: 分布式变量, AKA future/自动async

## tasks

> function call

```python
import ray

@ray.remote
def func_call():
    return 1


def main():
    ray.init()

    result_handle = []
    for i in range(4):
        result_handle.append(func_call.remote())
    print([ray.get(h) for h in result_handle])

if __name__ == "__main__":
    main()
```

相当于

```python
import ray

def func_call():
    return 1

def main():
    result_handle = []
    for i in thread_pool(4):
        result_handle.append(func_call())
    print([future.get(h) for h in result_handle])
    ray.shutdown()


if __name__ == "__main__":
    main()
```

## actors

> 分布式对象/状态

- class.remote(): 创建分布式对象

```python
@ray.remote
class Counter:
    def __init__(self):
        self.count = 0

    def increment(self):
        self.count += 1
        return self.count

def main():
    ray.init()

    counter = Counter.remote()
    for i in range(4):
        print(ray.get(counter.increment.remote()))
    ray.shutdown()

if __name__ == "__main__":
    main()

```

## objects

> 分布式中间结果的存储

所有的结果都是object store

1. 获取结果需要ray.get()
2. 不需要等待计算完成就可以传递(async)
3. 可以手动创建object store

```python
handle1 = func_call.remote()
handle2 = func_call.remote(handle1)
custom_data = ray.put([1, 2, 3])
print(ray.get(handle2))
# 批量等待
ray.wait([handle1, handle2])
```

