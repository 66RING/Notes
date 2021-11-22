---
title: qemu中MemoryRegion管理的模型
author: 66RING
date: 2021-8-22
tags: 
- softmmu
- qemu
mathjax: true
---

# 任务

- qemu管理MR的数据结构, 为什么地址空间可以嵌套
	* x86 负责地址空间分配的芯片是如何模拟的
- 设计树状结构MR的目的，中间节点，叶子节点是什么。父子关系代表什么。以及为何这么设计
- 协助王，找到qemu是如何完成GPA -> HVA的转换的


# 树状结构的MemoryRegion

这章关注的重点是：设计树状结构MR的目的。包括中间节点，叶子节点是什么，父子关系代表什么，以及为何这么设计。

首先说明结论：

1. 非叶子节点都是container抽象，叶子节点是真正会被映射的MR
2. 这么设计可以加速地址地址空间映射的过程(即在适当的位置插入一段地址空间)
3. 父子关系一方面可以方便管理(如下文ccsr的例子)，另一方面将整个大的线性空间分成了数个小部分，加速地址空间映射(插入只需遍历小部分，而不用遍历整个大链表)

MR的映射由`memory_region_add_subregion()`完成，其将mr插入适当的位置后发起地址空间更新操作`memory_region_transaction_commit`。

```
memory_region_add_subregion_common -> memory_region_update_container_subregions
```

`mr->subregions`表示当前MR(container)的直接子MR链表。生成平坦空间(FlatView)就是遍历该链表完成的(后面详细说明)。插入代码如下，主要是在`QTAILQ_FOREACH`中根据优先级priority将待映射MR插入到`subregions`中，组织好`subregions`结构后进入下一步平坦空间生成

```c
static void memory_region_update_container_subregions(MemoryRegion *subregion)
{
    MemoryRegion *mr = subregion->container;
    MemoryRegion *other;

    memory_region_transaction_begin();

    memory_region_ref(subregion);
    QTAILQ_FOREACH(other, &mr->subregions, subregions_link) {
        if (subregion->priority >= other->priority) {
            QTAILQ_INSERT_BEFORE(other, subregion, subregions_link);
            goto done;
        }
    }
    QTAILQ_INSERT_TAIL(&mr->subregions, subregion, subregions_link);
done:
    memory_region_update_pending |= mr->enabled && subregion->enabled;
    memory_region_transaction_commit();
}
```

由以上代码可知MemoryRegion的树状结构大致如下：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/qemu/qemu_mr_management/MR_tree.png" alt="" width=100%>


平坦空间生成通过递归遍历`mr->subregion`，将所有叶子mr插入FlatView中。这个递归过程类似树的前序遍历，只是顺序是由priority决定的。

```c
memory_region_transaction_commit() -> generate_memory_topology() -> render_memory_region()

static void render_memory_region(FlatView *view, MemoryRegion *mr, Int128 base, AddrRange clip,
                                 bool readonly,
                                 bool nonvolatile)
{
    /* Render subregions in priority order. */
    QTAILQ_FOREACH(subregion, &mr->subregions, subregions_link) {
        render_memory_region(view, subregion, base, clip, 				// 递归
                             readonly, nonvolatile);
    }

    if (!mr->terminates) {
        return;
    }

	...
    fr.mr = mr;
    fr.dirty_log_mask = memory_region_get_dirty_log_mask(mr);
    fr.romd_mode = mr->romd_mode;
    fr.readonly = readonly;
    fr.nonvolatile = nonvolatile;

	flatview_insert(view, i, &fr);
	...
}
```

## 为什么非叶子节点都是container抽象

通过生成平坦空间的代码(`render_memory_region`)可以知道为什么非叶子节点都是container抽象:

```c
static void render_memory_region(FlatView *view, MemoryRegion *mr, Int128 base, AddrRange clip,
                                 bool readonly,
                                 bool nonvolatile)
{
    /* Render subregions in priority order. */
    QTAILQ_FOREACH(subregion, &mr->subregions, subregions_link) {
        render_memory_region(view, subregion, base, clip,
                             readonly, nonvolatile);
    }

    if (!mr->terminates) {
        return;
    }

	...
    fr.mr = mr;
    fr.dirty_log_mask = memory_region_get_dirty_log_mask(mr);
    fr.romd_mode = mr->romd_mode;
    fr.readonly = readonly;
    fr.nonvolatile = nonvolatile;

	flatview_insert(view, i, &fr);
	...
}
```

可以看到如果不是叶子节点(`!mr->terminates`)就直接return了，不会执行`flatview_insert()`插入平坦空间中，也就不会映射内存。


## 为什么这么设计(猜测)

qemu内存空间管理引入了MR优先级的概念，优先级低的mr可以被优先级高的mr覆盖，这就要求按照优先级次序进行映射。qemu这里先映射高优先级的mr，有空隙时才能低优先级mr才能映射，这样能够减少调用`mmap`系统调用的次数。

当要求优先级有序，新地址空间的插入(映射)就会产生麻烦。因为如果使用一个链表组织所有mr，那么新插入的mr需要遍历整个链表，比对其他mr的优先级才能找到自己的位置。如果采用树状mr + 部分链表`subregions`的方式就可以加速mr插入的过程。


```c
static void memory_region_update_container_subregions(MemoryRegion *subregion)
{
    MemoryRegion *mr = subregion->container;
    MemoryRegion *other;

    memory_region_transaction_begin();

    memory_region_ref(subregion);
    QTAILQ_FOREACH(other, &mr->subregions, subregions_link) { 				// 遍历链表找合适的位置插入
        if (subregion->priority >= other->priority) {
            QTAILQ_INSERT_BEFORE(other, subregion, subregions_link);
            goto done;
        }
    }
    QTAILQ_INSERT_TAIL(&mr->subregions, subregion, subregions_link);
done:
    memory_region_update_pending |= mr->enabled && subregion->enabled;
    memory_region_transaction_commit();
}
```


## MR叠加的例子

MR叠加本质上也是一种分层管理，使用时只需要关系局部的细节，这在指定区域有固定偏移的情况会比较好用。powerpc的ccsr机制就是一个很好的例子。

ccsr是管理设备的一段地址区域，每种类型的设备在这段区域中都有固定的偏移，而ccsr这段这段区域的位置是灵活可变的。即设备在ccsr中的相对位置不变，在整个地址空间的绝对位置灵活可变。

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/qemu/qemu_mr_management/ccsr_usage.png" alt="" width=100%>

这样，首先根据需求为ccsr映射地址空间(1)，其他设备就可以根据在ccsr中的相对位置映射到ccsr空间(2)

```
1.
    memory_region_add_subregion(address_space_mem, pmc->ccsrbar_base,
                                ccsr_addr_space);

2. 
	/* I2C */
    memory_region_add_subregion(ccsr_addr_space, MPC8544_I2C_REGS_OFFSET,
                                sysbus_mmio_get_region(s, 0));
    /* General Utility device */
    memory_region_add_subregion(ccsr_addr_space, MPC8544_UTIL_OFFSET,
                                sysbus_mmio_get_region(s, 0));

```


# GPA 到 HVA的过程

qemu总是从tlb中找索引，再利用索引找mr

**不是完整的状态机**

1. 指令经过qemu动态翻译后得到GPA地址字面值，qemu需要根据该地址找到对应的MR然后执行相应的读写操作
2. GPA地址字面值在`XXX_helper`(如`load_helper`)中先查tlb，在利用tlb查找MemoryRegionSection(MR section)
	- 如果tlb没命中，则先`tlb_fill() -> -> tlb_set_page_with_attrs()`更新tlb。如果命中，则返回`iotlbentry`(TODO：iotlbentry只是一种情况)用于下一阶段section的查找
	- 在`tlb_set_page_with_attrs() -> address_space_translate_for_iotlb()`中会执行`phys_page_find()`查找section，计算出section的地址更新tlb，之后查tlb就能查到section了
	```c
	void tlb_set_page_with_attrs(CPUState *cpu, target_ulong vaddr,
								 hwaddr paddr, MemTxAttrs attrs, int prot,
								 int mmu_idx, target_ulong size)
	{
		section = address_space_translate_for_iotlb(cpu, asidx, paddr_page,
													&xlat, &sz, attrs, &prot);
		if (is_ram) {
			iotlb = memory_region_get_ram_addr(section->mr) + xlat;
			...
		} else {
			iotlb = memory_region_section_get_iotlb(cpu, section) + xlat;
			...
		}

		desc->iotlb[index].addr = iotlb - vaddr_page;
		desc->iotlb[index].attrs = attrs;
	}
	```
3. 经过`XXX_helper`，得到`tlbentry`，真正的读写将通过`tlbentry`查找MR section从而找到MemoryRegion完成访存操作

查找`MemoryReginoSection`的过程，主要发生在查找tlb中，因为tlb命中后直接用命中的`iotlbentry`来索引整个线性空间查找section。如下面`io_readx`这个例子，直接用tlbentry来索引整个线性空间就找到了section(`return &sections[index & ~TARGET_PAGE_MASK]`)

```c
static uint64_t io_readx(CPUArchState *env, CPUIOTLBEntry *iotlbentry,
                         int mmu_idx, target_ulong addr, uintptr_t retaddr,
                         MMUAccessType access_type, MemOp op)
{
    section = iotlb_to_section(cpu, iotlbentry->addr, iotlbentry->attrs);
	...
}

MemoryRegionSection *iotlb_to_section(CPUState *cpu,
                                      hwaddr index, MemTxAttrs attrs)
{
    int asidx = cpu_asidx_from_attrs(cpu, attrs);
    CPUAddressSpace *cpuas = &cpu->cpu_ases[asidx];
    AddressSpaceDispatch *d = qatomic_rcu_read(&cpuas->memory_dispatch);
    MemoryRegionSection *sections = d->map.sections;

    return &sections[index & ~TARGET_PAGE_MASK];
}
```

接下来将介绍究竟是如何找到MR section来填充tlb的。首先需要介绍几个结构:`AddressSpaceDispatch`, `PhysPageEntry`, `PhysPageMap`

```c
struct PhysPageEntry {
    /* How many bits skip to next level (in units of L2_SIZE). 0 for a leaf. */
    uint32_t skip : 6;
     /* index into phys_sections (!skip) or phys_map_nodes (skip) */
    uint32_t ptr : 26;
};

typedef PhysPageEntry Node[P_L2_SIZE];

typedef struct PhysPageMap {
    struct rcu_head rcu;

    unsigned sections_nb;
    unsigned sections_nb_alloc;
    unsigned nodes_nb;
    unsigned nodes_nb_alloc;
    Node *nodes;
    MemoryRegionSection *sections;
} PhysPageMap;

struct AddressSpaceDispatch {
    MemoryRegionSection *mru_section;
    /* This is a multi-level map on the physical address space.
     * The bottom level has pointers to MemoryRegionSections.
     */
    PhysPageEntry phys_map;
    PhysPageMap map;
};
```

qemu维护了一个类似多级页表的结构用于检索MR section。在上述结构中`PhysPageEntry`相当于页表项，`PhysPageMap`用于维护整个地址空间。

- PhysPageMap

`PhysPageMap`中的`nodes`是用于检索的多级页表结构; `sections`表示被索引的整个地址空间，索引到section后(tlbentry)直接`sections[index]`获取section

- AddressSpaceDispatch

`AddressSpaceDispatch`是一段地址空间的入口，`mru_section`是利用局部性原理的section缓存; `phys_map`是第一级表(`nodes`)的索引; `map`是内存映射结构的引用

整体关系大致如下图

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/Notes/universe/qemu/qemu_mr_management/phymap.png" alt="" width=100%>

基本结构了解后可以看看查找MR section的核心函数`phys_page_find`

```c
static MemoryRegionSection *phys_page_find(AddressSpaceDispatch *d, hwaddr addr)
{
    PhysPageEntry lp = d->phys_map, *p; 				// 获取第一级表的索引
    Node *nodes = d->map.nodes;
    MemoryRegionSection *sections = d->map.sections; 	// 整个线性空间
    hwaddr index = addr >> TARGET_PAGE_BITS;
    int i;

    for (i = P_L2_LEVELS; lp.skip && (i -= lp.skip) >= 0;) {
        if (lp.ptr == PHYS_MAP_NODE_NIL) {
            return &sections[PHYS_SECTION_UNASSIGNED];
        }
        p = nodes[lp.ptr];
        lp = p[(index >> (i * P_L2_BITS)) & (P_L2_SIZE - 1)]; 	// 一级一级查表
    }

    if (section_covers_addr(&sections[lp.ptr], addr)) {
        return &sections[lp.ptr]; 						// 利用最后一级表的值索引整个线性空间，返回section
    } else {
        return &sections[PHYS_SECTION_UNASSIGNED];
    }
}
```

因此，tlbentry就相当于记录了直接到`sections`的索引，加速了查表速度，获取到section后就可以`section->mr`获取到对应的MemoryRegion，从而完成访存操作




# 草稿

## tmp

- 如何GPA -> HVA
- 根据动态翻译到的GPA地址字面值先找到MR section，再用section找到MR，又因为MR绑定对应到一段ramblock:TODO

1. qemu动态翻译得到一个GPA地址字面值称addr
2. **IM**解析该addr找到MR section: `io_writex() -> iotlb_to_section(cpu, iotlbentry->addr, iotlbentry->attrs)`
	- 就是flatview 到 mr 的过程
	- hint: "section是注册到KVM的基本单位"
	- TODO: 找section的具体过程
3. 从section找到MR`mr = section->mr`，这样有了`mr`, `mr_offset`(addr字面值+tlb的东西，TODO)，操作`op`等TODO
4. 正式开始读写`r = memory_region_dispatch_write(mr, mr_offset, val, op, iotlbentry->attrs);`
	- `memory_region_dispatch_write -> mr->ops->write_with_attrs() -> 如:subpage_write()`
	```c
	static MemTxResult subpage_write(void *opaque, hwaddr addr,
									 uint64_t value, unsigned len, MemTxAttrs attrs)
	{
		subpage_t *subpage = opaque;
		uint8_t buf[8];

	#if defined(DEBUG_SUBPAGE)
		printf("%s: subpage %p len %u addr " TARGET_FMT_plx
			   " value %"PRIx64"\n",
			   __func__, subpage, len, addr, value);
	#endif
		stn_p(buf, len, value);
		return flatview_write(subpage->fv, addr + subpage->base, attrs, buf, len);
	}
	```

所有section组织在`AddressSpaceDispatch`中

## 填AddressSpaceDispatch

`flatview_add_to_dispatch -> register_subpage -> subpage_register`等将section写入

- `AddressSpaceDispatch *d = flatview_to_dispatch(fv);`
	* **`phys_section_add()`**


## 填"页表"

`phys_page_set`

`../accel/tcg/cputlb.c -> tlb_fill() -> x86_cpu_tlb_fill()`

## 找

- TODO lookup
	* `phys_page_find`
	
TODO: 以上三者的关系
- AddressSpaceDispatch: 没miss直接找??
- set/lookup: miss的时候重写AddressSpaceDispatch??
- 为什么这么设计
	* 模拟softmmu、tlb??
		+ 使可以直接GVA -> HVA??

- TODO
- 为何要从section到mr，而不直接找到mr

## TODO 

hint: `io_readx()/io_writex()/get_page_addr_code().`

- `load_helper`等


# MR树

- 设计树状结构MR的目的，中间节点，叶子节点是什么。父子关系代表什么。以及为何这么设计
	* 父子关系: 父: 抽象container，子: 抽象container/region

- 以container为单位更新


链表

```c
static void memory_region_update_container_subregions(MemoryRegion *subregion)
{
    MemoryRegion *mr = subregion->container;
    MemoryRegion *other;

    memory_region_transaction_begin();

    memory_region_ref(subregion);
    QTAILQ_FOREACH(other, &mr->subregions, subregions_link) { 		for (other = sb->first; other; other=other->next)
        if (subregion->priority >= other->priority) { 					<=================== ???为何subregion开遍?
            QTAILQ_INSERT_BEFORE(other, subregion, subregions_link);
            goto done;
        }
    }
    QTAILQ_INSERT_TAIL(&mr->subregions, subregion, subregions_link);
done:
    memory_region_update_pending |= mr->enabled && subregion->enabled;
    memory_region_transaction_commit();
}
```


`memory_region_transaction_commit()`更新, 无参->全局，怎么全局?

1. `flatviews_reset()`生成一段段，存入hash表，待用
2. foreach AS `address_space_set_flatview()`查表，组装
- 为什么这么设计?
	* section相关?
	* `memory_region_get_flatview_root`


还有这么的结构!!本质上还是用链表来更新

```
		MR
	/		\
   mr1  ->   mr2 

```


```
memory_region_transaction_commit -> flatviews_reset

static void flatviews_reset(void)
{
    AddressSpace *as;

    if (flat_views) {
        g_hash_table_unref(flat_views);
        flat_views = NULL;
    }
    flatviews_init();

    /* Render unique FVs */
    QTAILQ_FOREACH(as, &address_spaces, address_spaces_link) {
        MemoryRegion *physmr = memory_region_get_flatview_root(as->root);

        if (g_hash_table_lookup(flat_views, physmr)) {
            continue;
        }

        generate_memory_topology(physmr);
    }
}
```

`MemoryRegion *physmr = memory_region_get_flatview_root(as->root);`??TODO

```c
static MemoryRegion *memory_region_get_flatview_root(MemoryRegion *mr)
{
    while (mr->enabled) {
        if (mr->alias) {
            if (!mr->alias_offset && int128_ge(mr->size, mr->alias->size)) {
                /* The alias is included in its entirety.  Use it as
                 * the "real" root, so that we can share more FlatViews.
                 */
                mr = mr->alias;
                continue;
            }
        } else if (!mr->terminates) {
            unsigned int found = 0;
            MemoryRegion *child, *next = NULL;
            QTAILQ_FOREACH(child, &mr->subregions, subregions_link) {
                if (child->enabled) {
                    if (++found > 1) {
                        next = NULL;
                        break;
                    }
                    if (!child->addr && int128_ge(mr->size, child->size)) {
                        /* A child is included in its entirety.  If it's the only
                         * enabled one, use it in the hope of finding an alias down the
                         * way. This will also let us share FlatViews.
                         */
                        next = child;
                    }
                }
            }
            if (found == 0) {
                return NULL;
            }
            if (next) {
                mr = next;
                continue;
            }
        }
        return mr;
    }
    return NULL;
}
```

`generate_memory_topology`

```c
static FlatView *generate_memory_topology(MemoryRegion *mr)
{
    int i;
    FlatView *view;

    view = flatview_new(mr);

    if (mr) {
        render_memory_region(view, mr, int128_zero(),
                             addrrange_make(int128_zero(), int128_2_64()),
                             false, false);
    }
    flatview_simplify(view);

    view->dispatch = address_space_dispatch_new(view);
    for (i = 0; i < view->nr; i++) {
        MemoryRegionSection mrs =
            section_from_flat_range(&view->ranges[i], view);
        flatview_add_to_dispatch(view, &mrs);
    }
    address_space_dispatch_compact(view->dispatch);
    g_hash_table_replace(flat_views, mr, view);

    return view;
}

static void render_memory_region(FlatView *view,
                                 MemoryRegion *mr,
                                 Int128 base,
                                 AddrRange clip,
                                 bool readonly,
                                 bool nonvolatile)
{
    MemoryRegion *subregion;
    unsigned i;
    hwaddr offset_in_region;
    Int128 remain;
    Int128 now;
    FlatRange fr;
    AddrRange tmp;

    if (!mr->enabled) {
        return;
    }

    int128_addto(&base, int128_make64(mr->addr));
    readonly |= mr->readonly;
    nonvolatile |= mr->nonvolatile;

    tmp = addrrange_make(base, mr->size);

    if (!addrrange_intersects(tmp, clip)) {
        return;
    }

    clip = addrrange_intersection(tmp, clip);

    if (mr->alias) {
        int128_subfrom(&base, int128_make64(mr->alias->addr));
        int128_subfrom(&base, int128_make64(mr->alias_offset));
        render_memory_region(view, mr->alias, base, clip,
                             readonly, nonvolatile);
        return;
    }

    /* Render subregions in priority order. */
    QTAILQ_FOREACH(subregion, &mr->subregions, subregions_link) {
        render_memory_region(view, subregion, base, clip,
                             readonly, nonvolatile);
    }

    if (!mr->terminates) {
        return;
    }

    offset_in_region = int128_get64(int128_sub(clip.start, base));
    base = clip.start;
    remain = clip.size;

    fr.mr = mr;
    fr.dirty_log_mask = memory_region_get_dirty_log_mask(mr);
    fr.romd_mode = mr->romd_mode;
    fr.readonly = readonly;
    fr.nonvolatile = nonvolatile;

    /* Render the region itself into any gaps left by the current view. */
    for (i = 0; i < view->nr && int128_nz(remain); ++i) {
        if (int128_ge(base, addrrange_end(view->ranges[i].addr))) {
            continue;
        }
        if (int128_lt(base, view->ranges[i].addr.start)) {
            now = int128_min(remain,
                             int128_sub(view->ranges[i].addr.start, base));
            fr.offset_in_region = offset_in_region;
            fr.addr = addrrange_make(base, now);
            flatview_insert(view, i, &fr);
            ++i;
            int128_addto(&base, now);
            offset_in_region += int128_get64(now);
            int128_subfrom(&remain, now);
        }
        now = int128_sub(int128_min(int128_add(base, remain),
                                    addrrange_end(view->ranges[i].addr)),
                         base);
        int128_addto(&base, now);
        offset_in_region += int128_get64(now);
        int128_subfrom(&remain, now);
    }
    if (int128_nz(remain)) {
        fr.offset_in_region = offset_in_region;
        fr.addr = addrrange_make(base, remain);
        flatview_insert(view, i, &fr);
    }
}
```

!!!!非叶子直接return, 说明叶子是TODO

- 嵌套例子: `ccsr`
	* 但是映射只映射叶子节点，??从而父节点起到管理作用

- 叶子节点做映射，树枝节点做路由(映射用)

> **(leaves) are RAM and MMIO regions, while other nodes represent
> buses, memory controllers, and memory regions that have been rerouted**.

- 树状间接链表好处
	* 简化优先级??TODO
	* 优化插入，只需要遍历+比对 **子链表**，而不需要整个链，然后生成好后最后遍历一一边就能映射好了
- 直接链表不好
	* 插入O(N)，链表每个节点都有比如`start`，那么插入的时候就要遍历+比对

```
1.
    memory_region_add_subregion(address_space_mem, pmc->ccsrbar_base,
                                ccsr_addr_space);

2. 
	/* I2C */
    memory_region_add_subregion(ccsr_addr_space, MPC8544_I2C_REGS_OFFSET,
                                sysbus_mmio_get_region(s, 0));
    /* General Utility device */
    memory_region_add_subregion(ccsr_addr_space, MPC8544_UTIL_OFFSET,
                                sysbus_mmio_get_region(s, 0));

```

```
    if (!mr->terminates) {
        return;
    }

```

## [pass] render

TODO： 根据subregion一段线性空间，部分更新用。生成的flatview存到哈希表，最后最后组装(`address_space_set_flatview`)再用

!!ASSERT
- 递归调用, FILO, TODO
- 优先级高的在链表前
- `/* Render the region itself into any gaps **left by** the current view. */`

1. 从root开始，递归调用`render_memory_region`将mr映射为扁平空间的一段region
- "见缝插针" TODO: 流程

## [pass] flatview_add_to_dispatch

映射到内存`register_XXXpage -> phys_page_set`


```
                                                        PhysPageEntry[]
                                                             nodes
                                 (uint ptr, skip)          ┌───────┐      ┌───────┐
AddressSpaceDispatch      ┌──────────────────────────+──►  │ node  │      │ node  │
                          │                          │     ├───────┤(ptr) ├───────┤ (ptr)
        ┌──────────┐      │                          │     │ node  │ ───► │ node  │ ────┐
        │ phys_map ├──────┘     PhysPageMap          │     ├───────┤      ├───────┤     │
        ├──────────┤                map              │     │ node  │      │ node  │     │
        │   map    ├───────────►┌──────────┐         │     ├───────┤      ├───────┤     │
        └──────────┘            │  nodes   ├─────────┘     │       │      │       │     │
                                ├──────────┤               │ ....  │      │ ....  │     │
                                │ sections ├─┐             │       │      │       │     │
                                └──────────┘ │             │       │      │       │     │
                 sections                    │             └───────┘      └───────┘     │
                                             │                                          │
                 ┌───────┐                   │                                          │
                 │section│  ◄────────────────+──────────────────────────────────────────┘
                 ├───────┤
                 │section│
                 ├───────┤
                 │section│
                 ├───────┤
                 │ ....  │
                 └───────┘
```
