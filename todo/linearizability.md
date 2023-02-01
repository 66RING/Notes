# 一致性简述

## 线性一致性

- 相当于只有一个副本, 且操作是原子的

是全序(total order)的, 全序的意思的序列中的任意两个数都是可比较的。

## 因果一致性

区别于线性一致性, 因果一致性只要求存在因果的事件顺序唯一, 不存在因果的事件可以任意。比如同时发生的互不相关的事件, **再比如git的两个分支**

是偏序(partial order)的, 如集合类型就是不可比较的, 如`{1, 4}, {2, 3}`怎么比


## 线性一致性读

> t1时刻的写在t1时刻后一定能读到

## zookeeper

非强一致性，只保证写的强一致性，不保证读的强一致性。因为读能下发到follower中执行，提升吞托量。

而raft之类的都是通过leader执行，从而导致leader成为瓶颈节点


## raft

写自然线性一致

**读需要额外保证**, 论文的方式

- read index
- lease read

- 一方面，leader会比follower"超前"，而这"超前"并不能保证应用，所以读只能在commit index处执行
- 另一方面，leader可能被分区到少数派

### read index

- leader
	* todo
- follower
	* todo

等状态机应用到read index

leader发心跳给follower，确认它还是leader

TODO的主要作用**保证leader仍是leader: 考虑老leader被分区到少数派里的情况。**新leader在多数派分区选出，老leader在少数派分区也意识不到，不会退位。如果在老leader应用，即少数派，则


### lease read

类似read index，但减少网络交互

思路: 






