---
title: cpp常用stl 
date: 2020-10-26
tags: 
- c/c++
- stl
mathjax: true
---

## 函数

### sort

- `<algorithm>`，头文件中
- `sort(begin, end, compare)`
    * 传入表示开始结束的**随机访问**迭代器
    * 第三个参数compare可选，是一个具有两个参数的bool值函数，默认用`<`比较，从小到大


## Queue

### 优先队列

`priority_queue`模板有三个参数，第一个参数是储存对象的类型，第二个参数是存储元素的底层容器，第三个参数是一个决定元素顺序的断言。其中第二参数默认`vector`，第三参数默认`<` **大的元素排在前面** 。

```cpp
template <typename T, typename Container=std::vector<T>, typename Compare=std::less<T>> class priority_queue
```

创建一个优先队列

```cpp
using namespace std;
priority_queue<string, vector<string>,greater<string>> words1
```

创建一个储存字符串的优先队列，其底层存储结构是`vector`，根据`>`判断顺序。

由于默认顺序是按照`<`比较，所以也可在结构体中重载`<`运算符，在实现自定义顺序的目的

```cpp
struct node{
    int dis;
    int vi;
    bool operator <( const node &x )const
    {
        return x.dis < dis;
    }
};
priority_queue<node> q;
```
