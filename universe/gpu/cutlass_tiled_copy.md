---
title: cutlass tiled copy的本质
author: 66RING
date: 2025-11-28
tags:
- gpu
- hpc
mathjax: true
---

# Cutlass Tiled Copy

> Copy is all you need.


`make_tiled_copy`语义理解。核心在于: `tiler`和`layout_tv`。先说结论: 用atom去对tv layout进行分tile。用tiler去对目标tensor进行分tile。最后将这两个layout组合得到新的tv layout，表示tile-wise的访问tv, v的layout能够保证满足tiler的逻辑切分。

1. 我们想要连续访存, 所有用atom去tile`layout_tv`
2. 我们想要分块计算, 所有用tiler去tile目标tensor
3. 当用Tiler进行分tile后, tv中的v的布局将方法改变, 所以最后体现的是一个新的tv layout, 其中的v的layout是tiler后的逻辑layout

快速上手(Atom-wise访问tv):

```
  //  (Thr,(FrgV,FrgX),(RestM,RestN,...))
  //   Thr:   The logical threads within the tiled copy.
  //   FrgV:  The values local to a COPY_ATOM Dst.
  //   FrgX:  The values tiled across COPY_ATOMs Dst.
  //   RestM: The values tiled in M.
  //   RestN: The values tiled in N.
```

- Terms
    - `zipped_divide(layout, tile) = (tile, rest)`
        - layout用tile取切并在tile内根据tile重新映射
    - `right_inverse(layout)`, 返回一个新layout`result`, 有`layout(result(i)) = i`
        - AKA: 如果原始映射会把`(0...n)`映射成`(x0...xn)`, 求一个映射: 当输入是`(x0...xn)`时输出是`(0...n)`
        - cute中所有的layout都是通过逻辑idx联系起来的, `new = right_inverse(a_layout)`可以理解成当new的输入的逻辑idx时输出是`a_layout`

```cpp
// 本质:
//  1. 对tv layout分组(一atom为一组)
//  2. 分tile将对调整tv中v的layout
//      即tv中的v进行分组
//  最终的语义
//  (grouped_thr, grouped_val)
//  where
//      grouped_val(atom_v, rest_v) is tiled by Tiler
//      grouped_val(atom_v, rest_v):(new tile stride)
void _test_tidfrg_D() {
  using copy_op = UniversalCopy<cute::uint32_t>;
  using copy_traits = Copy_Traits<copy_op>;
  using copy_atom = Copy_Atom<copy_traits, float>;

  // (24, 16)
  constexpr Tensor dtensor = make_identity_tensor(make_shape(Int<24>{}, Int<16>{}));
  constexpr auto thr_layout = make_layout(make_shape(Int<8>{}, Int<16>{}));
  constexpr auto val_layout = make_layout(make_shape(Int<2>{}, Int<4>{}));

  constexpr int tid = 0;
  auto tiled_copy = make_tiled_copy(copy_atom{}, thr_layout, val_layout);
  using TiledCopy = decltype(tiled_copy);

  // (16, 64) AKA (mn.shape[0], mn.shape[1])
  auto tiler_mn = TiledCopy::Tiler_MN{};

  // ((tiler_m, tile_n), (rest_m, rest_n))
  // ((16, 64), (2, 1))
  auto tiled_tensor =  zipped_divide(dtensor, TiledCopy::Tiler_MN{});
  // (1, 1):(0, 0)
  auto ref2trg = right_inverse(TiledCopy::AtomLayoutRef{}).compose(TiledCopy::AtomLayoutDst{});


  // zipped_divide(input_layout, tile) = (tile, rest)
  // Input:
  //    TiledLayout_TV = ((8, 16), (2, 4))
  // Tiler:
  //    AtomNumThr = 1
  //    AtomNumVal = 1
  //
  // atom_layout_TV = ((1, 1), ((8, 16), (2, 4)))
  // 用tv去切atom, 让tv layout以atom为单位分组
  auto atom_layout_TV = zipped_divide(TiledCopy::TiledLayout_TV{}, make_shape(TiledCopy::AtomNumThr{}, TiledCopy::AtomNumVal{}));

  // Transform to the trg layout
  // 但这里还是和atom_layout_TV相同
  // atom_layout_TV = ((1, 1), ((8, 16), (2, 4)))
  //           (atom_t, atom_v)  thr^     val^
  auto trg_layout_TV = atom_layout_TV.compose(ref2trg, _);

  // NOTE: Transform the thrs mode from thrid to thr_idx
  // ((1, (8, 16)), (1, (2, 4)))
  // (atom_t * thr, atom_v * val)
  // AKA (thr和thr组, val和val组)
  auto _step_zip0 = zip(trg_layout_TV);
  // coalesce把符合(_, (_, _))模式的mode合并, 如这里的
  //  (1, (8, 16))
  //  (1, (2, 4))
  // NOTE: AKA按逻辑组(atom)访问thr, val
  // thrval2mn = ((8, 16), (1, 2, 4))
  auto thrval2mn = coalesce(zip(trg_layout_TV), Shape<_1,Shape<_1,_1>>{});

  // Transform the tile mode
  // ((16, 64), (2, 1)) o ((8, 16), (1, 2, 4))
  //   thrval2mn(tid, vid) -> m, n
  //   tiled_tensor(m, n) -> idx
  // tv_tensor.shape =
  // tv_tensor = (((8, 16), (1, 2, 4)), (2, 1)):...
  auto tv_tensor = tiled_tensor.compose(thrval2mn, _);
  // Unfold and return
  // unfold = ((8, 16), (1, 2, 4), (2, 1)):...
  //          ( thr     ,Frg       ,Rest )
  auto unfold = tv_tensor(make_coord(_,_), _);

  // NOTE: 速通到unfold
  //  1. tv layout根据atom分组 -> atom_layout_TV
  //    tv_layout(tid, vid) -> (m, n)
  //    此时tid, vid的遍历以组(atom)为单位进行(g_offset, gid)
  //  2. 目标矩阵按tiler分组 -> tiled_tensor
  //    target_tensor(m, n) -> physical idx
  //  3. 用atom_layout_TV去取tiled后的tensor
  //    tv_layout(tid, vid) -> (m, n)
  //               tiled_tensor(m, n) -> physical idx
  //  4. 重整输入的视图(tid, vid)
  //    原本的tv layout(tid, vid)可以看成是tid持有几个vid。但现在是(1)经过aomt分组的 (2)mn又是tiled的
  //    重新整理视图(语义)
  //    (tid, vid)
  //     ^     ^
  //     |     |
  //  语义不变,| 但是by atom的访问
  //           |
  //           |
  //  分tile将影响value的布局, 但总可以抽象成(组value, Rest)
  //  而组value又可以分成(组内value, 组id)
  //  所以，最终输出
  //  (Thr,(FrgV,FrgX),(RestM,RestN,...))
  //   Thr:   The logical threads within the tiled copy.
  //   FrgV:  The values local to a COPY_ATOM Dst.
  //   FrgX:  The values tiled across COPY_ATOMs Dst.
  //   RestM: The values tiled in M.
  //   RestN: The values tiled in N.
  // TODO: 
  //  1. 还没有理解ref2trg的功能
}
```
