---
title: 为什么需要swizzle
author: 66RING
date: 2025-09-10
tags:
- gpu
- cutlass
mathjax: true
---

# 为什么需要swizzle

e.g.

- 转置
- ldmatrix

假设有4路bank, 列向能并行

## 转置

e.g. 把如下矩阵转置。

```
0 1 2 3
x x x x
```

此时4路bank只用了1路。bank conflict。

```
0 x
1 x
2 x
3 x
```

## ldmatrix

ldmatrix可以一次加载一个matrix。但是matrix的行和行之间不一定是连续的，所以会有bank conflict。

swizzle可以把matrix的不同行的列地址错开, 以减少bank conflict。

```
  Smem layout:
    MMA0 | MMA2
    ------------
    MMA1 | MMA3

  Smem layout:
   0 1 | 8  9
   2 3 | 10 11
   -------------
   4 5 | 12 13
   6 7 | 14 16

  NOTE:
  warp内conflict:
    0 1
    2 3
    只用了2路bank
  而
    0 1 x x
    x x 2 3
    可以用到4路bank
  同理扩展到更多维度
```

因为ldmatrix一次是传入一行(这里一行是2列元素)的地址的, 所以swizzle后再加载数据仍能保持一致。e.g.

swizzle前

```
0 1 x x
2 3 x x
addr0: 0 1
addr1: 2 3
```

swizzle后

```
0 1 x x
x x 2 3
addr0: 0 1
addr1: 2 3
```

## swizzle in action

```cpp
#include <cstdint>
#include <iostream>

using namespace std;

template <int CBits, int BBits, int SShift = CBits> struct MySwizzle {
  static constexpr int num_base_bits_ = BBits;
  static constexpr int num_col_bits_ = CBits;
  static constexpr int num_shft_bits_ = SShift;

  static constexpr uint32_t ccc_msk = ((1 << num_col_bits_) - 1);
  static constexpr uint32_t rrr_msk =
      (ccc_msk << (num_col_bits_ + num_base_bits_));

  template <class T> constexpr auto operator()(T const &offset) const {
    // NOTE: do swizzle: rrr ^ ccc
    return offset ^ ((offset & rrr_msk) >> num_shft_bits_);
  }
};

template <uint32_t N> struct BitWidth {
  static_assert(N != 0, "Number must be non-zero and a power of two");
  static constexpr int value = 1 + BitWidth<(N >> 1)>::value;
};

template <> struct BitWidth<1> {
  static constexpr int value = 0;
};

int main() {
  static constexpr int kTileK = 32;
  static constexpr int group_size = 8;
  static constexpr int col_bits = BitWidth<kTileK / group_size>::value;
  cout << col_bits << endl;
  auto swizzle = MySwizzle<col_bits, 3>{};

  for (int i = 0; i < 128 * 4; ++i) {
    if (i % 32 == 0) {
      cout << endl;
    }

    printf("%3d, ", swizzle(i) % 32);
  }
  return 0;
}
```

运行发现成功把一个matrix的加载在行上错开了

```
 0,   1,   2,   3,   4,   5,   6,   7,   8,   9,  10,  11,  12,  13,  14,  15,  16,  17,  18,  19,  20,  21,  22,  23,  24,  25,  26,  27,  28,  29,  30,  31, 
 8,   9,  10,  11,  12,  13,  14,  15,   0,   1,   2,   3,   4,   5,   6,   7,  24,  25,  26,  27,  28,  29,  30,  31,  16,  17,  18,  19,  20,  21,  22,  23, 
16,  17,  18,  19,  20,  21,  22,  23,  24,  25,  26,  27,  28,  29,  30,  31,   0,   1,   2,   3,   4,   5,   6,   7,   8,   9,  10,  11,  12,  13,  14,  15, 
24,  25,  26,  27,  28,  29,  30,  31,  16,  17,  18,  19,  20,  21,  22,  23,   8,   9,  10,  11,  12,  13,  14,  15,   0,   1,   2,   3,   4,   5,   6,   7, 
....
```

原本要load的matrix是`0, ..., 7`的这个4x8

```
 0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
 0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
 0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
 0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
....
```

swizzle后要load的matrix是还是`0, ..., 7`的这个4x8, 但是每行的列地址错开了

```
0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
x,   x,   x,   x,   x,   x,   x,   x,   0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x, 
x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   0,   1,   2,   3,   4,   5,   6,   7,   x,   x,   x,   x,   x,   x,   x,   x, 
x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   x,   0,   1,   2,   3,   4,   5,   6,   7, 
```
