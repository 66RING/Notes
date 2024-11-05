---
title: Intel AVX
author: 66RING
date: 2024-11-04
tags: 
- system
- architecture
- arch
mathjax: true
---

# Intel AVX

> [Intel® Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) is all you need.

- SSE: Streaming SIMD Extensions
- AVX: Advanced Vector Extensions

- flow(类似于GPU编程)
    1. 创建SIMD寄存器
    2. 加载数据到SIMD寄存器
    3. 使用SIMD指令
    4. 将SIMD寄存器中的数据存回内存

## Terms

- **命名格式**
    * 数据类型命名: `__m` + SIMD寄存器位宽 + 数据类型(不加字母表示单精度), e.g.
        + `__m256`表示256位宽的单精度浮点数, `__m256i`表示256位宽的整数, `__m256d`表示256位宽的双精度浮点数
        + CPU需要**先把数据加载到专门的寄存器**才能进行向量指令操作
    * Intrinsic函数命名: `_mm` + 数据类型 + 操作名 + 操作范围和数据类型, e.g.
        + `_mm`默认128位宽, `_mm256`, `_mm512`
        + 操作名如: `_add`, `_mul`, `_load`, `_loadu`等
            + `_loadu`表示不对齐加载
        + 操作范围和数据类型如: `_ps`, `_pd`, `_epi32`等
            + `_ps`packed(操作所有数据)single(单精度)
            - `_ss`single(操作单个数据)single(单精度)
                * 第一个元素, 最低位元素
            - e.g.
                * `_mm256_load_ps`表示将浮点数加载到256位宽的寄存器
                * `_mm256_add_ps`表示将整个向量相加
- 万能头: `<intrin.h> `
- 常用头
    * `<immintrin.h>` Intel-specific intrinsics(AVX)
- 编译配置
    * `-msseN`, `-mavxN`其中N表示版本号


### 向量加法

```cpp
#define EPR 4 // elements per register, 128/32=4
vector<float> a(100000, 1.0f);
vector<float> b(100000, 1.0f);
vector<float> c(100000, 1.0f);

__m128 ra;
__m128 rb;
__m128 rc;
for (int i = 0; i < a.size(); i += EPR) {
    ra = _mm_loadu_ps(&a[i]);
    rb = _mm_loadu_ps(&b[i]);
    rc = _mm_add_ps(ra, rb);
    _mm_storeu_ps(&c[i], rc);
}
```

使用非内存对齐版本的load和store是因为创建变量时没保证内存对齐。这时使用内存对齐的load和store可能会触发边界保护异常。


### 内存操作

创建内存对齐的变量

- MSVC: `__declspec(align(32)) float a[8];`
- GCC: `__attribute__((__aligned__(32))) float a[8];`
- 动态分配:
    * `_aligned_malloc(size, 32)`或`new ((std::align_val_t)32) float[SIZE]`, 并使用对应的`_aligned_free`释放


`_mm_load_ps`, `_mm_store_ps`的本质就是指针类型转换并接引用。

```cpp
/* Load four SPFP values from P.  The address must be 16-byte aligned.  */
extern __inline __m128 __attribute__((__gnu_inline__, __always_inline__, __artificial__))
_mm_load_ps (float const *__P)
{
  return *(__m128 *)__P;
}

/* Store four SPFP values.  The address must be 16-byte aligned.  */
extern __inline void __attribute__((__gnu_inline__, __always_inline__, __artificial__))
_mm_store_ps (float *__P, __m128 __A)
{
  *(__m128 *)__P = __A;
}
```





