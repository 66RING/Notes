# page cache与缓存管理

直接从磁盘访问文件会很慢，可以利用空闲的内存来缓存一些磁盘文件的内容，这部分用作缓存磁盘文件的内存就叫做page cache。


## 1 execv中的调用page cache的流程

执行`read()`系统调用后，首先会查看page cache里有没有目标文件的内容，如果有(cache hit)，直接读取;如果没有(cache miss)再从磁盘上读取，放入page cache中。

由于内存空间有限，page cache也采用按需掉页(demand paging)的策略，需要时再申请内存把文件部分内容缓存起来。

每打开一个文件都会生成一个代表这个文件的`struct file`，但是文件的`struct inode`只有一个，inode才是文件的唯一标识，称为文件的owner。inode中含有指向`address_space`的指针。`address_space->a_ops`中包含一系列page和文件交互的方法，如`readapge()`，`writepage()`等。

TODO: address space

每个page都有其对应的文件，通过address space找到对应文件的操作方法。

```c
struct address_space {
        struct inode            *host;          /* owner: inode, block_device */
        struct radix_tree_root  i_pages;        /* cached pages */
        const struct address_space_operations *a_ops;   /* methods */
        ...
}
```

cache page采用radix tree结构存储，内核中使用`find_get_page`方法(对`pagecache_get_page`的封装)查找page。如果没找到就触发page fault，调用`__page_cache_alloc`在分配page，然后调用`add_to_page_cache(_lru)`加入page cache中。


## 2 页面替换算法:LRU(Least Recently Used)

LRU算法会换出最久未被使用的page。原理：根据程序的局部性原理，如果数据最近被访问过，那么它将来还被访问的概率也很高。

linux中用两个双向链表来实现lru，一个active list，一个inactive list。如果page最近被使用，则在active list中，否则在inactive list中。两个链表都是采用FIFO原则，最近最久未被使用的page将在列表尾部，因此内存回收时会从inactive list尾部page进行回收。

`struct page`的page flags使用`PG_referenced`和`PG_active`两个标志位来识别页面活跃程度。

`PG_active`为1表示在active list，`PG_active`为0表示在inactive list。

如果active list末尾page的`PG_referenced`为1则放回表头并置0;如果active list末尾page的`PG_referenced`为0，则表示最近未被使用，放入inactive list。

如果inactive list末尾page的`PG_referenced`为0，表示在inactive list期间中未被使用过，因此回收。如果inactive list中page的`PG_referenced`为2，则放回active list。如果末尾page的`PG_referenced`为1，会根据不同机制做不同的策略。


### 2.1 lru cache

可见page在active list和inacitve list中的切换是很频繁的，对于lru lock的竞争将会非常激烈。因此引入per-CPU的**lru cache**机制(使用`struct pagevec`表示)，换入换出的page先放在当前CPU的lru cache中，lru cache积累了`PAGEVEC_SIZE`个页面，再获取锁，批量转移。


### 2.2 refault

一个page被回收后再次被访问。因此分配这个page是通过page fault产生的，回收后再次被访问到又会触发的的page fault，此时的page fault称为 **refault**

如果inactive list足够长，那一个inactive list中的page在回收前有更多的时间可能被访问到，从而减少refault的发生。

当然由于内存有限，active list和inacitve list的长度需要权衡。


## 3 与kswapd的交互

> kswapd是linux中用于页面回收的内核线程。

内存严重不足时需分配内存，需要回收一部分内存(如用来做page cache的内存)后才能分配给新的需求。直接启动的内存回收操作叫做**direct reclaim**。这种模式下内存分配会被内存回收操作阻塞，等待时间长。而很多page的回收往往伴随着IO操作，如dirty页的写回，这就加剧了等待时间。

kswapd是linux中一种常见的回收机制，可以在内存压力不大时提前回收内存，维持空闲内存余量以减少内存压力大时发生缺页异常的负担。这种方式的内存回收又称为 **background reclaim** 

每个node对应一个kswapd内核线程，用于处理swap out、page cache等的回收和平衡acitve list和inacitve list等的任务。kswapd会调用`balance_pgdat()`来执行这些操作。

调用流程如下：

```
balance_pgdat
    kswapd_shrink_node
        shrink_node
            shrink_node_memcg
                shrink_list
                    if(inactive_list_is_low)
                        shrink_active_list
                    else
                        shrink_inactive_list
                            shrink_page_list
```

当`inactive_list_is_low`inactive list中的page较少时，`shrink_active_list`将active list尾端的一些page移动到inactive list。inactive list的长度增加可以减少refault的概率。

当inactive list中的page不是较少时。`shrink_inactive_list()->shrink_page_list()`从inactive list尾部移除选定数目的page进行回收。


### 3.1 watermark

至于什么时候触发kswapd，什么时候触发direct reclaim有watermark决定。

当空闲内存大于一定值时kswapd就会休眠。



