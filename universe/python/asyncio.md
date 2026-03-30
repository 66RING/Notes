---
title: python异步编程cheat sheet
author: 66RING
date: 2025-11-26
tags: 
- python
mathjax: true
---

# python异步编程

本质: (1)创建协程后后台执行, 还是(2)创建协程后"等待"执行。两者抽象出了asyncio的语法糖

1. `async def async_func()`可以快速定义异步方便
2. `await async_func()`会等待函数执行完成才继续下面的函数
    - 实际上`await`是主动挂起协程, 但还是在事件循环中运行
3. `task = asyncio.create_task(async_func(args))`才会真正异步, "后台执行"
    - 后台启动后立刻返回一个handle(task/future/可以等待对象)
    - `res = await task`主动等待完成, 或者使用`res = task.result()`
4. `asyncio.run(main)`启动入口函数

```python
import asyncio
import time

async def slow_task(name, delay):
    print(f"任务 {name} 开始")
    await asyncio.sleep(delay)  # 模拟耗时操作
    print(f"任务 {name} 结束")
    return f"{name} 结果"

async def main():
    # 情况1：await 调用——当前协程挂起，等任务完成
    print("=== 用 await 调用 ===")
    t = time.time()
    result1 = await slow_task("A", 1)
    print(time.time() - t)
    print(f"拿到结果: {result1}")  # 必须等A完成才执行这行

    # 情况2：create_task 创建后台任务——不等待，直接返回Task
    print("\n=== 用 create_task 后台执行 ===")
    t = time.time()
    task_b = asyncio.create_task(slow_task("B", 2))
    task_c = asyncio.create_task(slow_task("C", 1))
    print(time.time() - t)
    print("已创建B、C任务，继续执行主线程逻辑...")  # 立即执行，不等B/C完成

    # 后续可通过 await task_b 主动等待结果，或用 task_b.result()（需确保完成）
    t = time.time()
    result_b = await task_b
    result_c = await task_c
    print(time.time() - t)
    print(f"最终结果: {result_b}, {result_c}")

asyncio.run(main())

"""
=== 用 await 调用 ===
任务 A 开始
任务 A 结束
1.0012452602386475
拿到结果: A 结果

=== 用 create_task 后台执行 ===
4.792213439941406e-05
已创建B、C任务，继续执行主线程逻辑...
任务 B 开始
任务 C 开始
任务 C 结束
任务 B 结束
2.001450300216675
最终结果: B 结果, C 结果
"""
```

