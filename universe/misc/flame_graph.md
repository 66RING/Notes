---
title: 绘制火焰图
author: 66RING
date: 2022-01-01
tags: 
- misc
mathjax: true
---

# On-CPU 性能分析

On-CPU即"CPU密集型"

```
$ git clone https://github.com/brendangregg/FlameGraph
$ cd FlameGraph
$ sudo perf record --call-graph=dwarf <cmd>
$ sudo perf script | ./stackcollapse-perf.pl > out.perf-folded
$ ./flamegraph.pl out.perf-folded > perf.svg
```

使用浏览器打开`perf.svg`

# Off-CPU 性能分析

On-CPU即"IO密集型", 等花费在等待上(off cpu)的情况

```
$ git clone https://github.com/brendangregg/FlameGraph
$ cd FlameGraph
$ sudo offcputime-bpfcc -df -p `pgrep -nx mytest` 3 > out.stacks
$ ./flamegraph.pl --color=io --title="Off-CPU Time Flame Graph" --countname=us < out.stacks > out.svg
```

# ref

- https://rustmagazine.github.io/rust_magazine_2021/chapter_11/rust-profiling.html?search=
