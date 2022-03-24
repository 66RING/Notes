# Zookeeper

牺牲一定线性一致性，"特殊的线性一致性"，只保证写的一致性，读的线性一致性可以"手动同步"保证

# Chain replication

head -> n1 -> ... -> tail

每次从头传到尾replica，然后tail负责响应(read only from tail)，当tail出现问题时，tail的前一个顶替。

- **CR分担了主节点的压力**，因为负载分散到了个个`->`中
- 读只会涉及一个server(tail)，不像raft要广播
- 错误处理更加简单

TODO 线性一致性问题 lecture http://nil.csail.mit.edu/6.824/2020/notes/l-craq.txt

> TODO 有了raft为何还chain replication? 因为可以分region，你总是会需要一种办法来处理region之间的关系。然后region内部可以根据实际情况选择不同的策略


