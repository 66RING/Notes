---
title: title
author: 66RING
date: 2025-04-06
tags: 
- tags
mathjax: true
---

# A good slide

https://drive.google.com/file/d/1HU9O-B9Ycm-wlHS6vKxKFO7lEIXXBjfQ/view

# WGMMA

> https://research.colfax-intl.com/cutlass-tutorial-wgmma-hopper/

**tCsA isn’t actually a thread-level slice of SMEM**, but rather the entire SMEM tensor with a reorganized layout

## TiledMMA object

- WGMMA要求B矩阵必须在smem
- TiledMMA对象: `TiledMMA tiled_mma = cute::make_tiled_mma(SM90_64x64x16_F16F16F16_SS<GMMA::Major::MN,GMMA::Major::MN>{});`
    * 构造底层ptx: `wgmma.mma_async.sync.aligned.m64n64k16.f16.f16.f16`
    * Terms: `SM90_MxNxK_XYZ_SS or SM90_MxNxK_XYZ_RS`
        + X, Y, Z表示ABC矩阵的数据类型
        + MNK表示tile的大小
            + [list of allowed shapes: ](https://docs.nvidia.com/cuda/parallel-thread-execution/index.html#asynchronous-warpgroup-level-matrix-shape)
            + M is always 64, N is a multiple of 8 from 8 to 256, and for 16-bit operand datatype, K is 16 (more generally, K is fixed to be 32 bytes)
        + RS or SS表示AB矩阵的数据来源R(reg), S(smem)
        + MN, NT表示AB矩阵的存储顺序
            + MN表示行主序存储, NT表示列主序存储, K-major
    * thread个数: `dim3 dimBlock(cute::size(tiled_mma));`


## SMEM layout constraints

- 必须是MNK的倍数, 再和Swizzle等配置兼容, 所以可以用`tile_to_shape`自动计算
- `cute::tile_to_shape(Atom, Shape)`构造合适的shape
    * sA为例: `auto sA = cute::tile_to_shape(GMMA::Layout_MN_SW128_Atom<T>{}, cute::make_shape(bM, bK, bP));`
        + 其中`Atom = (64,8):(1,64)`, `(bM, bN, bK, bP) = (128, 128, 64, 3)`, `Sw<3,4,3>`
        + 最终结果`Sw&lt;3,4,3> o smem_ptr[16b](unset) o ((_64,_2),(_8,_8),_3):((_1,_512),(_64,_1024),_8192)`
            + 根据stride可知"iter"的先后顺序为`((1, 3), (2, 4), 5)`
        + bP表示pipeline个数
        + swizzle先不管
            + Swizzle<B,M,S>的BMS描述
                + 2^B: 几行一个单元(二维空间中有几行)
                + 2^M: 最小单元数
                + 2^S: 一行几个单元(二维空间中有几列)
            + 所以Swizzle<3,4,3>表示
                + 2^3 = 8行一个irow间隔
                + 2^4 = 16B一个单元
                + 2^3 = 8列, 16*8 = 128B
            + TODO: https://zhuanlan.zhihu.com/p/671419093
        + SW128表示128B一组, 因为`64 * sizeof(half_t) = 128`
        + 目标是把atom(64, 8)排成(128, 64, 3), 又是MN模式的, 所以64先排2次(_64, _2), 8排8次(_8, _8)

```
# sA, sB
Sw&lt;3,4,3> o smem_ptr[16b](unset) o ((_64,_2),(_8,_8),_3):((_1,_512),(_64,_1024),_8192)
```

## WGMMA Fragments and Descriptors


```
tCsA: Sw&lt;3,4,3>_smem_ptr[16b](0x7f8800000400) o ((_64,(_8,_2)),_2,_4,_3):((_1,(_64,_1024)),_512,_2048,_8192)
tCsB: Sw&lt;3,4,3>_smem_ptr[16b](0x7f880000c400) o ((_64,(_8,_2)),_2,_4,_3):((_1,(_64,_1024)),_512,_2048,_8192)
tCrC: ptr[16b](0x7feee1fffbe0) o ((_2,_2,_8),_2,_2):((_1,_2,_4),_32,_64)
```

- **thread layout: (MMA,MMA_M,MMA_K,PIPE)**
    * MMA: 表示Atom的shape
        + `(M, K) = (_64,(_8,_2)) = (_64,_16)`
    * `MMA_M`, `MMA_K`表示tile在M mode, K mode的扩展
        + MMA_M=bM/64=2, MMA_K=bK/16=4
    * **tCsA isn’t actually a thread-level slice of SMEM**, but rather the entire SMEM tensor with a reorganized layout
- C矩阵(输出)的thread layout: tCrC (MMA,MMA_M,MMA_N)
    * atom = (M, N) = (64, 64)
    * 表示MMA atom’s MxN=64x64 output tile
    * MMA: 表示每个thread负责的`32=2*2*8`个values
        + 拆成`(2,2,8)`是为了兼容定义好的模式[from the PTX documentation](https://docs.nvidia.com/cuda/parallel-thread-execution/index.html#wgmma-64n16-d)
            + `((2,2,8), 2, 2)`后面的2, 2和tCsA, tCsB一样
        + **TODO**: 如何和ptx和图例对应?
        + e.g. `wgmma.mma_async.m64nNk16`, N有一些可选的值
    * `MMA_M`, `MMA_N`同上


TODO: review


### gemm call

```cpp
// (V,M) x (V,N) => (V,M,N)
cute::gemm(tiled_mma, tCrA(_,_,_,read_pipe), tCrB(_,_,_,read_pipe), tCrC);
```

- 每个tile处理V个value, 最终gemm重载会reduce到`(V)x(V)=>(V)`
    * https://github.com/NVIDIA/cutlass/blob/be60a0b27204078dc0f3f1d6ed4a95cdb2114111/include/cute/algorithm/gemm.hpp#L178


## WGMMA matrices

- A矩阵可以在smem和reg, B矩阵只能在smem


# TMA


# Misc

coord映射的约定**列主序映射**: 先安排列, 再安排行: `(logical_thr_id, logical_val_id) -> (m, n) == m + n * M`, 即index从第一列开始排

**Since CuTe layouts return indices rather than coordinates, we choose a column-major encoding of the (m,n) coordinates**

```cpp
// e.g
template <class D, class A, class B, class C>
struct MMA_Traits<UniversalFMA<D,A,B,C>>
{
  using ValTypeD = D;
  using ValTypeA = A;
  using ValTypeB = B;
  using ValTypeC = C;

  // Logical shape of the MMA
  using Shape_MNK = Shape<_1,_1,_1>;

  // Logical thread id (tid) -> tidx
  // NOTE: 逻辑id(0..N)到物理id(tid)的映射
  using ThrID   = Layout<_1>;

  // (Logical thread id (tid), Logical value id (vid)) -> coord
  // NOTE: 追外层的rank是(logical_tid, logical_vid), tid再细分的话表示tid怎么映射
  // logical_tid = (0..N), 需要用ThrID来映射到物理id
  // (l_tid, l_vid)映射到一个元素的index(coord)
  // (logical_thr_id, logical_val_id) -> (m, n) == m + n * M

  // (tid,vid) -> (m,k)
  using ALayout = Layout<Shape<_1,_1>>;
  // (tid,vid) -> (n,k)
  using BLayout = Layout<Shape<_1,_1>>;
  // (tid,vid) -> (m,n)
  using CLayout = Layout<Shape<_1,_1>>;
};
```

An MMA_Traits specialization defines the following public type aliases.

- ValTypeD: Logical compute type of the D matrix
- ValTypeA: Logical compute type of the A matrix
- ValTypeB: Logical compute type of the B matrix
- ValTypeC: Logical compute type of the C matrix
- Shape_MNK: Logical MxNxK shape of the MMA operation
- ThrID: Logical thread mapping within the single MMA operation (specifying the thread, quadpair, warp, or warpgroup view)
- ALayout: Mapping of (thread,value) pairs to coordinates in the MxK A matrix
- BLayout: Mapping of (thread,value) pairs to coordinates in the NxK B matrix
- CLayout: Mapping of (thread,value) pairs to coordinates in the MxN C matrix



## API

### Tensor

> https://zhuanlan.zhihu.com/p/1892344162487104421

- `make_tensor`
- `make_counting_tensor`, 创建一个tensor, Tensor的值是layout的offset
- `make_identity_tensor`, 创建一个tensor, Tensor的值是对应的坐标
- slice: e.g. `a(2, _)`相当于python中的`a[2, :]`
- 分区
    * 内部分区(inner-partitioning): 处理一整个block
        + `local_tile`
        + `inner_partition`, 和`local_tile`互为别名
        + 一般做thread block级区分, 分区到不同block上
    * 外部分区(outer-partitioning): 处理block中的一个元素
        + `local_partition`
        + `outer_partition`, 和`local_tile`互为别名
        + 一般做thread级区分, 分区到不同thread上
    * 线程-值分区(Thread-Value partitioning)
        + 第一维表示thread layout, 第二维表示(一个thread的)value layout: (thread, value) -> (m, n)
            + **使用thread id索引tv layout可以得到thread需要处理的元素**
        + 复合函数操作composition: offset = Layout(coordinate)
            + (g o f)(x) = g(f(x))
            + index映射成(m, n), 一个layout处理完(m, n)后再将结果转成index如此往复
                + see [02_layout_algebra](https://github.com/NVIDIA/cutlass/blob/v3.8.0/media/docs/cute/02_layout_algebra.md#composition)
            + 多维layout的复合: TODO



