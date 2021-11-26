---
title: QEMU中GPA到HVA变换过程
author: 66RING
date: 2021-9-26
tags: 
- softmmu
- qemu
mathjax: true
---

# GPA 到 HVA的过程

QEMU通过GPA找到MemoryRegion从而完成到HVA的转换，而QEMU总是先从tlb中找tlbentry，再利用tlbentry找mr。GPA到HVA的问题就变为了，GPA找tlbentry的问题。

**不是完整的状态机**

1. 指令经过qemu动态翻译后得到GPA地址字面值`addr`，qemu需要根据该地址找到对应的MR然后执行相应的读写操作
2. GPA地址字面值在`XXX_helper`(如`load_helper`)中先查tlb，在利用查到的tlbentry索引出MemoryRegionSection(MR section)
	- 如果tlb没命中，则先`tlb_fill() -> -> tlb_set_page_with_attrs()`更新tlb。如果命中，则返回`tlbentry`(TODO：iotlbentry只是一种情况)用于下一阶段section的查找
	- 在`tlb_set_page_with_attrs() -> address_space_translate_for_iotlb()`中完成具体tlb更新操作。会执行`phys_page_find()`查找section，计算出section的地址更新tlb，之后再查tlb就有对应的entry索引section了
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

