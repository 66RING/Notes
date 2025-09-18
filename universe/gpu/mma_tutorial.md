---
title: Tensor core MMA指令教程
author: 66RING
date: 2025-04-09
tags:
- gpu
- hpc
mathjax: true
---

# Tensor core MMA指令教程

> 参考 https://zhuanlan.zhihu.com/p/1892346599864238276

以[mma.m8n8k4](https://docs.nvidia.com/cuda/parallel-thread-execution/index.html?highlight=mma%2520sync%2520aligned%2520m8n8k4#warp-level-matrix-fragment-mma-884-f16)为例

> A warp executing mma.m8n8k4 with .f16 floating point type will compute 4 MMA operations of shape .m8n8k4.

一个warp可以同时执行4个m8n8k4的MMA。买个MMA相互独立, 不reduce。

- MMA1: Threads with %laneid 0-3 (low group) and 16-19 (high group)
- MMA2: Threads with %laneid 4-7 (low group) and 20-23 (high group)
- MMA3: Threads with %laneid 8-11 (low group) and 24-27 (high group)
- MMA4: Threads with %laneid 12-15 (low group) and 28-31 (high group)

不同MMA组负责各自m8n8k4矩阵的计算, 最后通过thread value的layout把矩阵元素的位置还原。

MMA使用流程

1. 找到A B矩阵的TVlayout, 然后把元素传入对应的thread中
    - e.g. Figure 45 MMA .m8n8k4 fragment layout for row-major matrix A with
    - MMA1中T0-T3负责前4行, T16-T19负责后4行
    - 映射公式一般图片下面会给出
    ```
    (row, col)表示元素在矩阵中的位置, 需要搬运到thread的寄存器中
    i 表示thread中的value的index
    row =            %laneid % 4          if %laneid < 16
                    (%laneid % 4) + 4     otherwise

    col =            i                    for ai where i = {0,..,3}
    ```
2. 检查限定符, 如`.col.row`等
3. ptx调用
    - 文档会说明寄存器要如何传入, 例如两个f16要pack到一起
    - 这里m8n8k4的A矩阵的thread要传入4个f16, B矩阵的thread要传入4个f16, C矩阵的thread要传入8个f16
```cuda
    asm volatile("mma.sync.aligned.m8n8k4.row.col.f32.f16.f16.f32"
                 "{%0,  %1,  %2,  %3,  %4,  %5,  %6,  %7},"
                 "{%8,  %9},"
                 "{%10, %11},"
                 "{%12, %13, %14, %15, %16, %17, %18, %19};\n"
                 : "=f"(d[0]), "=f"(d[1]), "=f"(d[2]), "=f"(d[3]),
                   "=f"(d[4]), "=f"(d[5]), "=f"(d[6]), "=f"(d[7])
                 : "r"(a[0]), "r"(a[1]),
                   "r"(b[0]), "r"(b[1]),
                   "f"(c[0]), "f"(c[1]), "f"(c[2]), "f"(c[3]),
                   "f"(c[4]), "f"(c[5]), "f"(c[6]), "f"(c[7]));
```
4. 结果位置还原
    - e.g. Figure 50 MMA .m8n8k4 computation 1 and 2 fragment layout for matrix C/D with 
    - 计算完成后thread value的对应位置如图, 需要存储到C矩阵中
    - 图片下方会给出位置映射公式
```
row =     X           if %laneid < 16
        X + 4         otherwise

          where X = (%laneid & 0b1) + (i & 0b10)  for ci where i = {0,..,7}

col = (i & 0b100) + (%laneid & 0b10) + (i & 0b1)  for ci where i = {0,..,7}
```

ptx指令规则: 1. 顺序placeholder 2. 寄存器类型(处理r表示regular/u32, 其他都是常见的缩写)

```
"h" = .u16 reg  -> half
"r" = .u32 reg  -> regular/u32
"l" = .u64 reg  -> long
"q" = .u128 reg -> quad
"f" = .f32 reg  -> float
"d" = .f64 reg  -> double
```

## 完整代码

```cpp
// bang!:run:term make && %:p:h/build/%:t:r
// bang!:run:term %:p:h/build/%:t:r

#include <cstdio>
#include <thrust/host_vector.h>
#include <thrust/device_vector.h>
#include <cute/tensor.hpp>

using namespace cute;

template <class TA, class TB, class TC>
__global__ void mma_kernel_tn(TA *A, TB *B, TC *C, int M, int N, int K)
{
    uint32_t a[2];
    uint32_t b[2];
    float c[8] = {0.0f};
    float d[8] = {0.0f};

    int tid = threadIdx.x;

    int row_a = 0, col_b = 0;
    // 该8x8x4的MMA1有T0~T3+T16~T19组成, 其中T0~T3处理前4行, T16~T19处理后4行
    if (tid < 16)
    {
        row_a = tid % 4;
        col_b = tid % 4;
    }
    else
    {
        row_a = tid % 4 + 4;
        col_b = tid % 4 + 4;
    }

    // 一个uint32保留两个half
    a[0] = *reinterpret_cast<uint32_t *>(A + row_a * 4 + 0);
    a[1] = *reinterpret_cast<uint32_t *>(A + row_a * 4 + 2);
    b[0] = *reinterpret_cast<uint32_t *>(B + col_b * 4 + 0);
    b[1] = *reinterpret_cast<uint32_t *>(B + col_b * 4 + 2);

    asm volatile("mma.sync.aligned.m8n8k4.row.col.f32.f16.f16.f32"
                 "{%0,  %1,  %2,  %3,  %4,  %5,  %6,  %7},"
                 "{%8,  %9},"
                 "{%10, %11},"
                 "{%12, %13, %14, %15, %16, %17, %18, %19};\n"
                 : "=f"(d[0]), "=f"(d[1]), "=f"(d[2]), "=f"(d[3]),
                   "=f"(d[4]), "=f"(d[5]), "=f"(d[6]), "=f"(d[7])
                 : "r"(a[0]), "r"(a[1]),
                   "r"(b[0]), "r"(b[1]),
                   "f"(c[0]), "f"(c[1]), "f"(c[2]), "f"(c[3]),
                   "f"(c[4]), "f"(c[5]), "f"(c[6]), "f"(c[7]));

    // Figure 50 MMA .m8n8k4 computation 1 and 2 fragment layout for matrix C/D with
    // TV layout
    // row =   X           if %laneid < 16
    //         X + 4       otherwise
    //         where X = (%laneid & 0b1) + (i & 0b10)  for ci where i = {0,..,7}
    // col = (i & 0b100) + (%laneid & 0b10) + (i & 0b1)  for ci where i = {0,..,7}
    for (int i = 0; i < 8; ++i)
    {
        int row = (tid & 0b1) + (i & 0b10);
        if (tid > 16)
        {
            row += 4;
        }
        int col = (i & 0b100) + (tid & 0b10) + (i & 0b1);
        C[row * 8 + col] = d[i];
    }
}

int main()
{
    srand(1234);

    int M = 8, N = 8, K = 4;
    int A_size = M * K;
    int B_size = K * N;
    int C_size = M * N;

    using TA = cute::half_t;
    using TB = cute::half_t;
    using TC = float;

    thrust::host_vector<TA> h_A(A_size);
    thrust::host_vector<TB> h_B(B_size);
    thrust::host_vector<TC> h_C(C_size);

    for (int j = 0; j < A_size; ++j)
        h_A[j] = static_cast<TA>(rand() % 9 + 1);
    for (int j = 0; j < B_size; ++j)
        h_B[j] = static_cast<TB>(rand() % 9 + 1);
    for (int j = 0; j < C_size; ++j)
        h_C[j] = static_cast<TC>(0);

    thrust::device_vector<TA> d_A = h_A;
    thrust::device_vector<TB> d_B = h_B;
    thrust::device_vector<TC> d_C = h_C;

    dim3 blocks(1);
    dim3 threads(32);

    mma_kernel_tn<<<blocks, threads>>>(d_A.data().get(), d_B.data().get(), d_C.data().get(), M, N, K);

    // copy back
    thrust::copy(d_A.begin(), d_A.end(), h_A.begin());
    thrust::copy(d_B.begin(), d_B.end(), h_B.begin());
    thrust::copy(d_C.begin(), d_C.end(), h_C.begin());

    auto a_tensor = make_tensor(h_A.data(), make_shape(M, K), make_stride(K, 1));
    auto b_tensor = make_tensor(h_B.data(), make_shape(K, N), make_stride(1, K));
    auto c_tensor = make_tensor(h_C.data(), make_shape(M, N));
    print_tensor(a_tensor);
    print_tensor(b_tensor);
    print_tensor(c_tensor);

    return 0;
}
```
