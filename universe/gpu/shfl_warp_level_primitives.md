---
title: shfl, warp-level primitives
author: 66RING
date: 2024-08-11
tags: 
- gpu
- cuda
- hpac
mathjax: true
---

# shfl: warp-level primitives

一个warp有32个thread, warp内的线程称为通道(lanes), lane id的计算方法是`threadid % 32`, warp id的计算方法是`threadid / 32`。

线程束洗牌: warp-level原语

- 可以直接获取warp内的线程的寄存器值，**直接使用寄存器交换**
- 每次调用都会同步warp内的线程, `sync`
- mask用于指定需要参与计算的线程, bit mask的方式
    - tips `uint32_t(-1)`
- `__shfl_sync(unsigned mask, T var, int srcLane, int width=32)`
    - **广播交换**: 所有参与计算的线程都会获取到`srcLane`线程传入的var的值
- `__shfl_up_sync(unsigned mask, T var, unsigned delta, int width=32)`
    - **向上传递**: 参与计算的线程从自己的lane id - delta的线程获取var的值, 相当于i的值传递给i+delta的线程
    - 前delta个线程的值返回自身var的值
- `__shfl_down_sync(unsigned mask, T var, unsigned delta, int width=32)`
    - **向下传递**: 参与计算的线程从自己的lane id + delta的线程获取var的值, 相当于i的值传递给i-delta的线程
    - 前delta个线程的值返回自身var的值
- `__shfl_xor_sync(unsigned mask, T var, int laneMask, int width=32)`
    - **异或交换**: 参与计算的线程从自己的lane id ^ laneMask的线程获取var的值
- width: warp的大小, 默认32
    - 表示warp的逻辑大小, aka 几个warp做一组执行一次操作


## cheat sheet

经典用法

1. 广播: `__shfl_sync`
2. max reduce
    - `__shfl_up_sync`, `__shfl_down_sync`, 每次reduce一半, 直到结果汇聚到一个warp上
3. 数列求和:
    - `__shfl_up_sync`, e.g. 4加上shift上来的3 2 1
4. 通用reduce
    - `__shfl_xor_sync`, 两两交换数值, 递归reduce出结果

max reduce

```cpp
for (int i=16; i>=1; i/=2)
    value = max(__shfl_down_sync(0xffffffff, value, i, 32), value);
/*
   e.g.
   0 1 2 3  4 5 6 7
reduce
   0 1 2 3  4 5 6 7
   4 5 6 7  4 5 6 7
reduce
   4 5 6 7  |  4 5 6 7
   6 7 6 7  |  6 7 6 7
reduce
   6 7 | 6 7  |  6 7 | 6 7
   7 7 | 7 7  |  7 7 | 7 7
*/
```

一个通用reduce的实现

```cpp
#include <stdio.h>
#include <stdint.h>

template<typename T>
struct SumOp {
    __device__ inline T operator()(T const & x, T const & y) { return x + y;}
};

template<typename T>
struct MaxOp {
    __device__ inline T operator()(T const & x, T const & y) { return x > y ? x : y; }
};

template <int THREADS>
struct AllReduce {
    static_assert(THREADS == 32 || THREADS == 16 || THREADS == 8 || THREADS == 4);
    template <typename T, typename Operator>
    static __device__ T run(T x, Operator &op) {
        constexpr int OFFSET = THREADS / 2;
        x = op(x, __shfl_xor_sync(uint32_t(-1), x, OFFSET));
        return AllReduce<OFFSET>::run(x, op);
    }
};

// Specialization for 2 threads, which stops the recursion
template <>
struct AllReduce<2> {
    template <typename T, typename Operator>
    static __device__ T run(T x, Operator &op) {
        x = op(x, __shfl_xor_sync(uint32_t(-1), x, 1));
        return x;
    }
};

__global__ void warpReduce() {
    int laneId = threadIdx.x & 0x1f;
    int m = laneId;
    int s = laneId;

    // Sum
    SumOp<int> sumOp;
    s = AllReduce<32>::run(s, sumOp);

    MaxOp<int> maxOp;
    m = AllReduce<32>::run(m, maxOp);

    // "value" now contains the sum across all threads
    printf("Thread %d final max = %d, sum = %d\n", threadIdx.x, m, s);
}

int main() {
    warpReduce<<< 1, 32 >>>();
    cudaDeviceSynchronize();
    int sum = 0;
    for (int i = 0; i < 32; i++)
        sum += i;
    printf("Expected sum = %d\n", sum);

    return 0;
}
```

