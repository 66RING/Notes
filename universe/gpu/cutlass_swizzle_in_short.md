---
title: cutlass swizzle简单理解和实现
author: 66RING
date: 2025-09-08
tags:
- gpu
- cutlass
mathjax: true
---

# cutlass swizzle简单理解

> 物理行号=逻辑行号, 物理列号=逻辑列号^逻辑组号
>
> 一个index/offset有明显的组别关系, e.g. 0b rr ccc b, 表示|行号bits|列号bits|数据bits|

用swizzle映射一个逻辑地址, 可以实现无bank conflict的smem访问: `physical_addr = swizzle(logical_addr)`

证明:

1. 逻辑坐标(x, y), x表示行号, y表示列号
2. 行内无冲突证明: 一行横一定没有重复的id(不repeat的话)
3. 列内无冲突证明: 对于任意列(xl_i, y)不存在重复的id, y表示某一列, `xl_i`表示某一行的逻辑id
    - 列内任意两行`xl_i`, `xl_j`, 物理id分别为`xp_i = xl_i ^ y`, `xp_j = xl_j ^ y`
    - `xp_i ^ xp_j != 0`, 所以无冲突
        * `xp_i ^ xp_j = xl_i ^ xl_j ^ (y ^ y) = xl_i ^ xl_j ^ 0 != 0`


简单实现: 一个index转换成二进制表示可以看到明显的组别关系: 行号bits, 列号bits, 数据bits

```
e.g. 4行, 8列, 2B数据
index: 0b rr ccc b
rr表示行号所在bits
ccc表示列号所在bits
b表示数据所在bits

swizzle就相当于把ccc替换成ccc ^ rr
```

见cutlass中的实现:


## cutlass swizzle

根据上述原理和注释, 可以得出人话版`Swizzle<B, M, S>(offset)`的语义

- `Swizzle<B, M, S>(offset)`
    * M, MBase: 数据有效位, e.g. float4 -> 128bits
    * B, BBis:  列有效位
    * S, SShift:行号左移多少位可以和列号位置对齐
    * 当然BMS抽象是为了更通用实现, 不一定局限于行号列号, 我们只是方便理解

```cuda
// A generic Swizzle functor
/* 0bxxxxxxxxxxxxxxxYYYxxxxxxxZZZxxxx
 *                               ^--^ MBase is the number of least-sig bits to keep constant
 *                  ^-^       ^-^     BBits is the number of bits in the mask
 *                    ^---------^     SShift is the distance to shift the YYY mask
 *                                       (pos shifts YYY to the right, neg shifts YYY to the left)
 *
 * e.g. Given
 * 0bxxxxxxxxxxxxxxxxYYxxxxxxxxxZZxxx
 * the result is
 * 0bxxxxxxxxxxxxxxxxYYxxxxxxxxxAAxxx where AA = ZZ xor YY
 */
 template <int BBits, int MBase, int SShift = BBits>
struct Swizzle
{
  static constexpr int num_bits = BBits;
  static constexpr int num_base = MBase;
  static constexpr int num_shft = SShift;

  static_assert(num_base >= 0,             "MBase must be positive.");
  static_assert(num_bits >= 0,             "BBits must be positive.");
  static_assert(abs(num_shft) >= num_bits, "abs(SShift) must be more than BBits.");

  // using 'int' type here to avoid unintentially casting to unsigned... unsure.
  // NOTE: 数据位宽
  using bit_msk = cute::constant<int, (1 << num_bits) - 1>;
  // NOTE: "行号"位
  using yyy_msk = cute::constant<int, bit_msk{} << (num_base + max(0,num_shft))>;
  // NOTE: "列号"位
  using zzz_msk = cute::constant<int, bit_msk{} << (num_base - min(0,num_shft))>;
  using msk_sft = cute::constant<int, num_shft>;

  static constexpr uint32_t swizzle_code = uint32_t(yyy_msk::value | zzz_msk::value);

  template <class Offset>
  CUTE_HOST_DEVICE constexpr static
  auto
  apply(Offset const& offset)
  {
    // NOTE:
    // 1. shiftr(offset & yyy_msk{}, msk_sft{})
    //    shiftr表示右移
    //    offset & yyy_msk{} >> shift
    //    只保留yyy_bis, 其他bit清0
    //    yyy_bis >> shift
    //    000000yyy000 >> shift
    // 2. offset ^ (yyy_bis >> shift)
    //  0bxxxxxxxxxxxxxxxxYYxxxxxxxxxZZxxx
    //       xor
    //  0b000000000000000000000000000YY000
    return offset ^ shiftr(offset & yyy_msk{}, msk_sft{});   // ZZZ ^= YYY
  }

  template <class Offset>
  CUTE_HOST_DEVICE constexpr
  auto
  operator()(Offset const& offset) const
  {
    return apply(offset);
  }
};
```



