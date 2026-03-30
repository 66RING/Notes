## day

1. 如何generic

### ArrayString

`Vec<Option<String>>`不好，String是指针，缓存不好，随机读。数据要紧凑

改：

```
data:    233abc
         ^  ^  ^
         |  |  |--|
offsets: 0, 3, 6, 6
bitmap: true, true, false
```

String，Array"共存"

```
pub struct StringArray {
    /// The flattened data of string.
    data: Vec<u8>,

    /// Offsets of each string in the data flat array.
    offsets: Vec<usize>,

    /// The null bitmap of this array.
    bitmap: BitVec,
}

impl StringArray {
    fn get(&self, idx: usize) -> Option<&str> {
        if self.bitmap[idx] {
            let range = self.offsets[idx]..self.offsets[idx + 1];
            Some(unsafe { std::str::from_utf8_unchecked(&self.data[range]) })
        } else {
            None
        }
    }
}
```

### "generic associated types" (GAT)



## 表达式向量化


## 不同类型的比较

比如i32与i64比较






