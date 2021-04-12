---
title: QEMU与serial_mm_write交互过程
date: 2021-04-10
tags: 
- qemu
mathjax: true
---

# 线程模型

- QEMU中的主要线程
    * vCPU thread
        + 可根据不同加速器调用不同的vcpu创建函数，如对于kvm，会调用`qemu_kvm_start_cpu`
        + 虚拟机的每个CPU对应宿主机上的一个线程，通常叫做vCPU线程。vCPU线程用来执行虚拟机的代码
    * I/O thread
    * worker thread(VNC/SPICE)

早期的non-iothread模型，只存在一个主线程负责虚拟机指令执行和处理事件，显然无法并行处理多个事件，如果遇到磁盘读写等耗时很长的操作时，表现不佳。

因此，新版qemu使用新的架构：

- 为每个vCPU分配一个qemu线程
- 专用的事件处理循环线程，**iothread** 
    * iothread负责事件处理循环，使用全局的所互斥，来维持线程同步

各个vCPU线程就可以并行处理客户机指令。iothread阻塞时，vCPU仍可以运作客户机指令


# CPU虚拟化

## VMX架构

为了系统的稳定运行，会对指令进行分级，等级低的指令不应影响整个系统从而影响到同级别的程序。以x86CPU为例，x86CPU共有ring0-ring3，4个运行等级，操作系统运行在ring0，应用程序运行在ring3。

如果指令会影响到整个系统，则称指令是敏感指令，如读写时钟、读写中断寄存器等。如果指令只影响自身所在进程，则称指令是非敏感指令。当应用程序需要执行一些敏感操作，如访问系统资源时，需要特殊指令陷入内核，由内核进行一些列安全检查，然后执行操作。

同理为了虚拟机指令的独立和稳定，对指令进行划分。VMX定义了两类软件角色：VMM(虚拟机控制器)和VM(虚拟机)。

- VMM对各个VM进行管理，包括创建、配置、删除、分配资源等，以确保VM直接的隔离
    * VMM对整个系统的CPU和硬件有完全控制权，它抽象出虚拟CPU给各个VM。
- 每个VM是一个虚拟机实例，VM之间相互独立，有自己的CPU、内核、中断和设备等

VMM的运行权限高于VM，这样敏感操作需要到VMM中执行。

VMX执行的模式叫做VMX root operation，VM执行的模式叫做VMX non-root operation。从VMX root转换到VMX non-root的过程叫做 **VM Entry**，从VMX non-root转换到VMX root的过程叫做 **VM Exit**。

QEMU创建虚拟机线程，初始化的时候会设置好CPU寄存器的值，如果启用KVM，则可以在物理机上执行虚拟机的代码。如果虚拟机中执行的代码的敏感代码(即可能会影响到整体x系统)或满足一定退出条件是，CPU会从non-root模式退出到root模式(VM Exit)，交由KVM处理。如果KVM无法处理则分配给QEMU处理，当QEMU/KVM处理号事件后再将CPU置与VM non-root模式。虚拟机就这么不停的进行VM Exit和VM Entry。


## VCPU的运行

vcpu运行的核心函数是`cpu_exec()`(如果启用KVM，则是`kvm_cpu_exec()`)。其核心的一个循环，当虚拟遇到一些事件导致VM Exit时，就会交由KVM/QEMU处理。

如果启用kvm则，调用kvm的处理方式。如果没有启用KVM或KVM不可用，则指令或者说 **目标代码块(TB)** 由 **TCG(tiny code generator)** 翻译成宿主机架构的代码，然后由`cpu_tb_exec()`执行并修改虚拟机CPU到对应状态。


# 内存虚拟化

虚拟化软件需要提供如下过程，即所谓MMU虚拟化

```
虚拟机的虚拟地址
    |
    v
虚拟机的物理地址
    |
    v
QEMU的虚拟地址
    |
    v
物理地址
```


## 基本结构

- `AddressSpace`结构体：用来表示一个虚拟机或虚拟CPU能够访问的所有物理内存
    * `MemoryRegion *root`成员表示其对应的`MemoryRegion`
    * 其他子系统可以注册地址空间变更的事件，所注册的信息通过`listeners`连接
    * 所有`AddressSpace`通过`address_spaces_link`连接
    * HMP中输入`info mtree`可以看到所有`AddressSpace`，格式`address-space: devices`，表示设备视角下的内存空间
- `MemoryRegion`结构体：表示虚拟机的一段内存区域。内存的模拟是通过MemoryRegion结构体构成的无环图完成
    * 图的叶子节点是实际分配给虚拟机的物理内存或MMIO，中间节点表示总线
    * `ram_block`表示实际分配的物理内存
    * `ops`是一组回调函数
    * `container`表示当前`MemoryRegion`的上一级`MemoryRegion`
    * `subregion`连接当前`MemoryRegion`的子`MemoryRegion`
    * `addr`表示该`MemoryRegion`所在的虚拟机的物理地址
- `MemoryRegion`分类
    * RAM：**host上的一段虚拟内存**，做虚拟机的物理内存
    * MMIO：guest上的一段内存。host上没有对应的虚拟内存，会截获对这个区域的访问然后调用对应读写函数
    * ROM：与RAM类似，只读
    * ROM device：读方面类似RAM，直接读。写方面类似MMIO，调用对应回调函数
    * container：用于合并多个`MemoryRegion`，如PCI的`MemoryRegion`会包含RAM和MMIO
        + 包含若干个`MemoryRegion`，每个Region在这个container中的偏移不一样
    * alias：region的另一个部分，用于将region分成几个不连续部分


## 查找MemoryRegion的过程

每个cpu有一个`AddressSpace`的成员，可以由此找到对应`MemoryRegion`：

- 从`AddressSpace`的root成员开始，找到所有subregion，如果地址不再region中，则不考虑该region
- 如果地址在一个叶子`MemoryRegion`中，返回该region
- 如果子`MemoryRegion`是一个容器，则对该容器递归调用该算法
- 如果`MemoryRegion`是一个alias，则找对应的实际region开始
- 如果在一个容器或alias中没找到匹配项，且container本身有自己的MMIO和RAM，则返回这个容器本身，否则根据下一个优先级查找
- 如果未找到region，结束


## 内存初始化

调用`cpu_exec_init_all`

- `io_mem_init`
    * 创建若干个包含所有地址空间的`MemoryRegion`
- `memory_map_init`
    * 创建两个`AddressSpace`，和其对应的`MemoryRegion`。他们均为 **全局变量** 
        + `address_space`表示虚拟机的内存地址空间
        + `address_space_io`表示虚拟机的IO地址空间
- 分配虚拟机RAM
    * 最终会调用`memory_region_init_ram`完成分配，一些参数情况如下
        + `mr`表示RAM对应的`MemoryRegion`
        + `owner`表示所属的上级`MemoryRegion`
    * `memory_region_init_ram`中调用`memory_region_init`初始化`MemoryRegion`
        + 由于是初始化RAM(表示实际分配的内存)，所以`mr->ram`和`mr->terminates`设置为true
        + 之后`qemu_ram_alloc`分配以RAMBlock结构以及虚拟机物理内存对应的QEMU进程中的虚拟内存
        + RAMBlock表示虚拟机中的一块内存条
            + 包含各种信息：系统页大小`page_size`，已用大小`used_length`等
            + `offset`成员表示内存条在虚拟机整个内存中的偏移
            + 所有RAMBlock成员通过next连接
        + `ram_block_add`用来向系统添加新的内存条


# 虚拟机初始化

虚拟机初始化时，会先初始化两个全局的`AddressSpace`：`address_space_io`和`address_spaces_memory`，以及它们的`MemoryRegion`：`system_memory`和`system_io`，作为虚拟机的物理内存。

```
static MemoryRegion *system_memory;
static MemoryRegion *system_io;

AddressSpace address_space_io;
AddressSpace address_space_memory;
```

每个qom初始化时都有自己的`MemoryRegion`，设备通过`memory_region_add_subregion`直接或间接把自己的`MemoryRegion`添加到全局的`MemoryRegion`。虚拟机的物理内存本身也是一个qom，会在虚拟机创建时初始化，之后设备的添加再在虚拟机物理内存的基础上解析qemu参数，添加设备`MemoryRegion`

这样一来，任意设备就可以通过这全局的`AddressSpace`找到设备对应的`MemoryRegion`，从而找到执行对应的回调函数。

vcpu线程负责处理指令。如果启用了kvm，则要执行的指令交由kvm处理，否则QEMU会使用tcg将操作对应的指令转化翻译成目标结构下的指令(以下称TB，target block)。

即在未启用kvm，由qemu软件翻译tb的情况下，执行tcg下的主循环：

```c
../accel/tcg/cpu-exec.c

int cpu_exec(CPUState *cpu)
{
    /* if an exception is pending, we execute it here */
    while (!cpu_handle_exception(cpu, &ret)) {
        ...
        while (!cpu_handle_interrupt(cpu, &last_tb)) {
            ...
            tb = tb_find(cpu, last_tb, tb_exit, cflags);
            cpu_loop_exec_tb(cpu, tb, &last_tb, &tb_exit);
            ...
        }
    }
}
```

之后在`cpu_loop_exec_tb`中调用`cpu_tb_exec`执行tb。其中主函数是`tcg_qemu_tb_exec(CPUArchState *env, const void *tb_ptr)`。期间会调用`store_helper`或`load_helper`模拟mmu的tlb过程

之后实际内存读写，先找到对应的`MemoryRegionSection`从而找到对应的`MemoryRegion`，之后就可以找到对应从回调函数(称为accessor)


# 一个实例：serial\_mm\_write的访问过程

以openrisc架构下虚拟机处理串口输出为例。

以下是openrisc虚拟机初始化函数的主要流程：

```c
static void openrisc_sim_init(MachineState *machine)
{
    ...
    ram = g_malloc(sizeof(*ram));
    memory_region_init_ram(ram, NULL, "openrisc.ram", ram_size, &error_fatal);
    memory_region_add_subregion(get_system_memory(), 0, ram);
    ...
    serial_mm_init(get_system_memory(), 0x90000000, 0, serial_irq,
                   115200, serial_hd(0), DEVICE_NATIVE_ENDIAN);
    openrisc_load_kernel(ram_size, kernel_filename);
}
```

首先qemu申请一段内存作为虚拟机的物理内存(以下称为ram)`ram = g_malloc(sizeof(*ram))`，之后初始化ram`memory_region_init_ram`，然后将ram所代表的`MemoryRegion`通过`memory_region_add_subregion(get_system_memory(), 0, ram)`加入全局的的`MemoryRegion`。

虚拟机的物理内存准备好后接着初始化了`serial_mm`，并加入了全局的`MemoryRegion`

当你在虚拟机中按下键盘的瞬间，vcpu线程会感知到这个事件的发生。如果启用了kvm，则要执行的代码交由kvm处理，否则QEMU会使用tcg将操作对应的指令转化翻译成目标结构下的指令(以下称TB，target block)。

接下来就要处理内存，qemu会通过`load_helper`和`store_helper`模拟tlb过程。在`serial_mm_write`的访问过程中中会调用`io_writex`，`io_writex`主要内容如下：

```c
static void io_writex(CPUArchState *env, CPUIOTLBEntry *iotlbentry,
                      int mmu_idx, uint64_t val, target_ulong addr,
                      uintptr_t retaddr, MemOp op)
{
    ...
    section = iotlb_to_section(cpu, iotlbentry->addr, iotlbentry->attrs);
    mr = section->mr;
    ...
    // r 表示执行状态
    r = memory_region_dispatch_write(mr, mr_offset, val, op, iotlbentry->attrs);
    ...
}
```

可见`io_writex`会得到一个当前cpu，一个tlb实体，一个mmu索引，满足了操作内存所需的条件。这么就可以根据addr找对应的`MemoryRegionSection`从而找到`MemoryRegion`，之后调用`memory_region_dispatch_write`派发对应的写操作。

其中查找`MemoryRegionSection`的核心：`iotlb_to_section`的代码如下：

```c
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

每个cpu都有一个`AddressSpace`和对应的`AddressSpaceDispatch`，通过`AddressSpaceDispatch`成员就可以找到该cpu的`AddressSpace`对应的`MemoryRegionSection`，从而找到对应的`MemoryRegion`，从而可以找到对应的处理函数，即`serial_mm_write`



