---
title: Bitonic Sort
author: 66RING
date: 2023-12-29
tags: 
- algorithm
mathjax: true
---

# Bitonic Sort

> 只能用在序列长度是2的幂次
>
> 类似快排递归

- bitonic序列: 前一半单调递增, 后一半单调递减的序列
- bitonic pair
    * 任意的连个元素可以通过swap转换成bitonic序列
- 复杂度分析
    * 时间复杂度: O((logN)^2)
    * 空间复杂度: O(N(logN)^2)
- 优势: 并行度好
    * **每个pair/swap都可以并行处理**

algo

1. `bitonic_sort`, 分解整个序列成pair
    - **左右递归调用**, 从细粒度到粗粒度sort
2. `bitonic_merge`, 逐步合并成更大的bitonic序列
    - **依旧左右递归调用**, 从粗粒度到细粒度merge


伪代码

```c
// low: 起点index
// count: scope的大小, 当scope是1时说明全部处理完成, 终止
// direction: 升序/降序
// algo
// 1. TODO: review
void bitonic_sort(int[] arr, int low, int count, int direction) {
    if (count <= 1)
        return;

    int k = count / 2;
    // direction = 1, 升序
    bitonic_sort(arr, low, k, 1);
    // direction = 0, 降序
    bitonic_sort(arr, low + k, k, 0);
    bitonic_merge(arr, low, count, direction);
}

void bitonic_merge(int[] arr, int low, int count, int direction) {
    if (count <= 1)
        return;

    int k = count / 2;
    for (int i = low; i < low + k; i++) {
        // NOTE: 先merge再递归
        if (direction = (arr[i] > arr[i + k])) {
            swap(arr[i], arr[i+k]);
        }
        // NOTE: 先merge再递归
        // 先粗粒度merge, 再递归细粒度merge
        bitonic_merge(arr, low, k, direction);
        bitonic_merge(arr, low + k, k, direction);
    }

}
```


