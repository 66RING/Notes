---
title: GPU L2 Cache优化
author: 66RING
date: 2025-09-18
tags: 
- gpu
- hpc
mathjax: true
---

# GPU L2 Cache优化

In short: 1. 总体需要经手L2 Cache的数据少了，命中率就高了 2. 访问相同数据的thread block越多，命中率就高了

> https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html#l2-cache-optimizations
>
> L2 Cache是所有SM共享的cache

以gemm为例，假设一次最多调度4个thread block，用于计算C矩阵。

不优化的情况,  为了计算如下的C矩阵, 需要读取A, B矩阵总共5块。

```
   B B B B
A  0 1 2 3
   x x x x
         C
```

grouped优化(block id reorder)后, 只需要读取A, B矩阵总共4块。

```
   B B
A  0 1 x x
A  2 3 x x
         C
```

依旧是熟悉的Z字形排布

1. 需要cache的block少了(5->4)
2. 更倾向于访问相同的数据(e.g. 01访问相同的A, 02访问相同的B)

## 希尔伯特曲线, L2 Cache最优化

```
希尔伯特曲线
0 1 14 15
3 2 13 12
4 7  8 11
5 6  9 10
```

复用最优的情况一定是正方形的, e.g. A和B需要访问的block数量相同

因此, 可以用希尔伯特曲线(Hilbert curve)来将idx转换成tile id(坐标)。

但是边界问题很麻烦，所以还是用简单点的吧。（

> https://zhuanlan.zhihu.com/p/6872421770

```cpp
// https://github.com/lawmurray/gpu-gemm/blob/main/src/gemm.cu
template<int M, int N>
requires (M == N || M == N/2)
__device__ point2 hilbert2(const int s) {
  int i = 0, j = 0;
  int t = s;
  for (int k = 1; k < max(M, N); k *= 2) {
    int bi = 1 & (t/2);  // local gray code, u shape top left to bottom left
    int bj = 1 & (t ^ bi);
    if (bj == 0) {
      if (bi == 1) {
        i = k - 1 - i;  // flip up-down
        j = k - 1 - j;  // flip left-right
      }
      int tmp = i;  // transpose
      i = j;
      j = tmp;
    }
    i += k*bi;
    j += k*bj;
    t /= 4;
  }
  return {i, j};
}
```


