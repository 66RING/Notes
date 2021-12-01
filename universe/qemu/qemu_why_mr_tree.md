---
title: 关于qemu使用树状MemoryRegion的分析
author: 66RING
date: 2021-8-22
tags: 
- softmmu
- qemu
mathjax: true
---


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

## 结论

qemu中树状结构的mr并不是说地址空间可以嵌套，qemu树状结构mr的目的：一方面方便分组管理，另一方面这么做也体现了逻辑上的"多态"。比如说mmio是要映射到ram的，那这段被映射空间即是ram的，也是mmio的，表示"mmio映射到了ram上"，所以看上去是嵌套的关系。具体见下文MR作用3。

描述引用自[官方文档](https://github.com/coreos/qemu/blob/master/docs/memory.txt): https://github.com/coreos/qemu/blob/master/docs/memory.txt

**只需注意黑体加粗部分内容**

> Memory is modelled as an acyclic graph of MemoryRegion objects.  Sinks
> **(leaves) are RAM and MMIO regions, while other nodes represent
> buses, memory controllers, and memory regions that have been rerouted**.

1. MR树状结构作用1：方便分组管理

> - container: a container simply includes other memory regions, each at
>   a different offset.  Containers are **useful for grouping several regions
>   into one unit**.  For example, a PCI BAR may be composed of a RAM region
>   and an MMIO region.

2. MR树状结构作用2：如果一个mr没有handler，可以递归查找container，直到找到handler。有种缺省时的handler的感觉：mmio要映射到ram,如果mmio确实映射了，做mmio的handler,没有映射那就是做ram的handler(相当于访存)

> It is valid to add subregions to a region which is not a pure container
> (that is, to an MMIO, RAM or ROM region). This means that the region
> will act like a container, except that **any addresses within the container's
> region which are not claimed by any subregion are handled by the
> container itself** (ie by its MMIO callbacks or RAM backing).

3. MR树状结构作用3：方便通过mr的优先级(priority)机制，控制虚拟机(guest)能看到的内容，即控制mr叠加时那个mr可见。

- use case: mmio映射到ram上，而我们希望访问这段空间时将其视为mmio,使用mmio的handler而不是ram，那得让mmio的priority高于ram
	* 假设x86映射4GB ram到地址空间0x0-0x100000000(0-4GB)，又因为3.5G-4G是留给pci用的，里面模块化的为各个设备分配mmio。这段地址范围即是ram的，又是pci设备的，而我们想要pci设备的mmio可见，就可以提高pci设备mr的priority

> Overlapping regions and priority
> --------------------------------
> **Usually, regions may not overlap each other**; a memory address decodes into
> exactly one target.  In some cases it is useful to allow regions to overlap,
> and sometimes to **control which of an overlapping regions is visible to the
> guest.**  This is done with memory_region_add_subregion_overlap(), which
> allows the region to overlap any other region in the same container, and
> specifies a priority that **allows the core to decide which of two regions at
> the same address are visible (highest wins).**
> Priority values are signed, and the default value is zero. This means that
> you can use memory_region_add_subregion_overlap() both to specify a region
> that must sit 'above' any others (with a positive priority) and also a
> background region that sits 'below' others (with a negative priority).

下面是官方提供的一个内存映射的例子，可以大概看一下，注意就是为了体现上述内容。 用黑体标出了个人认为的关键点，并在其后面括号中加了说明：

**格式说明**：形如

```
 +---- vga-window: alias@0xa0000-0xbffff ---> #pci (0xa0000-0xbffff)
        (prio 1)
```

1. `vga-window`是该MemoryRegion的名称
2. `alias`是该MemoryRegion的类型，这里是`alias`就是说该MR是另一个MR的别名，别名MR是TODO
3. `@0xa0000-0xbffff`表示它在虚拟机视野(即GPA，即虚拟机物理地址)中的范围
4. `---> #pci (0xa0000-0xbffff)`，在2中说过，这个MR是一个别名MR，所以这里`--->`表示该MR的本体在pci这个container中，是表示pci中`0xa0000-0xbffff`这块区域的MR


> Example memory map
> ------------------
> 
> ```
> system_memory: container@0-2^48-1
>  |
>  +---- lomem: alias@0-0xdfffffff ---> #ram (0-0xdfffffff)
>  |
>  +---- himem: alias@0x100000000-0x11fffffff ---> #ram (0xe0000000-0xffffffff)
>  |
>  +---- vga-window: alias@0xa0000-0xbffff ---> #pci (0xa0000-0xbffff)
>  |      (prio 1)
>  |
>  +---- pci-hole: alias@0xe0000000-0xffffffff ---> #pci (0xe0000000-0xffffffff)
> 
> pci (0-2^32-1)
>  |
>  +--- vga-area: container@0xa0000-0xbffff
>  |      |
>  |      +--- alias@0x00000-0x7fff  ---> #vram (0x010000-0x017fff)
>  |      |
>  |      +--- alias@0x08000-0xffff  ---> #vram (0x020000-0x027fff)
>  |
>  +---- vram: ram@0xe1000000-0xe1ffffff
>  |
>  +---- vga-mmio: mmio@0xe2000000-0xe200ffff
> ```
> 
> ram: ram@0x00000000-0xffffffff
> 
> This is a (simplified) PC memory map. **The 4GB RAM block is mapped into the
> system address space via two aliases(体现作用1: 分组管理)**: "lomem" is a 1:1 mapping of the first
> 3.5GB; "himem" maps the last 0.5GB at address 4GB.  This leaves 0.5GB for the
> so-called PCI hole, that allows a 32-bit PCI bus to exist in a system with
> 4GB of memory.
> 
> The memory controller diverts addresses in the range 640K-768K to the PCI
> address space.  **This is modelled using the "vga-window" alias, mapped at a
> higher priority so it obscures the RAM at the same addresses.  The vga window
> can be removed by programming the memory controller(体现作用3：mr叠加时的可见性控制)**; **this is modelled by
> removing the alias and exposing the RAM underneath.(体现作用2：空隙时向上找到container的handler)**
> 
> The pci address space is not a direct child of the system address space, since
> we only want parts of it to be visible (we accomplish this using aliases).
> It has two subregions: vga-area models the legacy vga window and is occupied
> by two 32K memory banks pointing at two sections of the framebuffer.
> In addition the vram is mapped as a BAR at address e1000000, and an additional
> BAR containing MMIO registers is mapped after it.
> 
> Note that **if the guest maps a BAR outside the PCI hole, it would not be
> visible as the pci-hole alias clips it to a 0.5GB range.(体现作用1：分组管理)**

