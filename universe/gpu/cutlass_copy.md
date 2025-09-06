---
title: cutlass cute copy的本质
author: 66RING
date: 2025-09-06
tags: 
- gpu
mathjax: true
---

# cutlass cute copy的本质

本质: src_ptr和dst_ptr在**step时用的逻辑index进行for循环, 通过layout映射到物理index再进行访存。**

```cuda
template <class SrcEngine, class SrcLayout,
          class DstEngine, class DstLayout>
CUTE_HOST_DEVICE
void
copy(AutoCopyAsync                const& cpy,
     Tensor<SrcEngine, SrcLayout> const& src,
     Tensor<DstEngine, DstLayout>      & dst)
{
  for (int i=0; i < size(dst); ++i) {
    dst(i) = src(i);
  }
}
```

过程

1. 逻辑inndex i变化成坐标coord: `idx2crd(idx, shape, default_stride)`, `(i%shape[0], i/shape[0]%shape[1], ...)`
2. coord经过layout映射成物理index: `crd2idx(coord)`

tensor dst/src处理一个index时会经过内部记录的layout对齐进行转换再用于访存。

e.g. shape为`[8]`的tensor的拷贝

| 应用   | Src               | Dst               |
|--------|-------------------|-------------------|
| 1D拷贝 | `8:1`             | `8:1`             |
| ND拷贝 | `(1,2,3):(1,2,3)` | `(1,2,3):(1,2,3)` |
| 广播   | `8:0`             | `8:1`             |

广播的例子

```cuda
#include <cuda.h>
#include <cute/layout.hpp>
#include <cute/stride.hpp>
#include <cute/tensor.hpp>
#include <stdlib.h>

using namespace std;

using namespace cute;

int main() {
  int data[8] = {0, 1, 2, 3, 4, 5, 6, 7};
  auto src = make_tensor(make_gmem_ptr(data),
                         make_layout(make_shape(8), make_stride(0)));
  auto dst = make_tensor(make_gmem_ptr(data),
                         make_layout(make_shape(8), make_stride(1)));
  for (int i = 0; i < size(dst); i++) {
    cout << src(i) << "->" << dst(i) << "\n";
  }
  return 0;
}
```

输出

```
0->0
0->1
0->2
0->3
0->4
0->5
0->6
0->7
```

