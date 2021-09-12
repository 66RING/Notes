---
title: QEMU总线模拟
date: 2021-09-10
tags: 
- qemu
mathjax: true
---

# Abstract

设备之间通过总线互联通信，如果将所有设备都挂在到sysbus(即所谓的内部总线：连接cpu、内存、pic总线控制器等设备的总线)将会增加主板的布线难度，而且cpu并不需要实时与外围设备通信。因此没必要每个设备都直接与cpu连接。

现代cpu通过sysbus直接连接一个控制芯片，这个控制芯片再管理外围设备。控制芯片将数据汇总后再与cpu通信。

这个过程对应到QEMU中的就是：

- cpu通过sysbus找到pcihost
- pcihost通过pcibus找到pci设备完成cpu的指令

本文主要说明QEMU的怎么模拟实现这个过程的

# Preface

PCI设备有如下三种不同内存：

- MMIO
- PCI IO space
- PCI configuration space
- 
其中pci configuration space 是用来配置pci设备的，其中也包含了关于pci设备的特定信息。而config空间中的BAR: Base address register可以用来确定设备需要使用的内存或I/O空间的大小，也可以用来存放设备寄存器的地址。下文将对其进行详细说明


# 执行流程

QEMU中对sysbus的模拟就是将设备与MemoryRegion绑定，cpu执行指令，经过翻译后能够找到对应的MemoryRegion从而找到相应的设备。这里就是cpu找到pcihost的过程。

pcihost模拟是主要功能是：根据cpu的指令找到pci设备，然后"转发"指令给pci设备

pcibus模拟的主要功能是：维护pci设备信息，为pcihost索引目标设备提供依据

pci设备模拟的主要功能是：维护自己的config空间和操作回调函数，设备的内存空间映射将会根据config空间的内容完成

QEMU中的总线模拟将有pcibus模拟和pci设备模拟共同完成：pcibus提供设备的连接关系、pci设备提供config空间的信息

## 读写设备config空间

读写config空间可以分为两个阶段:

- 指令传输
- 指令执行

gdb调试的表现为：总是会先调用`pci_host_config_write`为`s->config_reg`赋值，然后会调用data read/write解析刚才传入的`s->config_reg`

<img src="./config_reg.png" alt="" width="100%">


### 指令传输

指令传输阶段的任务是将cpu的命令保存到`config_reg`中，方便在指令执行阶段根据pci协议规范解析命令。

```c
static void pci_host_config_write(void *opaque, hwaddr addr,
                                  uint64_t val, unsigned len)
{
    PCIHostState *s = opaque;

    PCI_DPRINTF("%s addr " TARGET_FMT_plx " len %d val %"PRIx64"\n",
                __func__, addr, len, val);
    if (addr != 0 || len != 4) {
        return;
    }
    s->config_reg = val;
}
```

可见将指令(`val`)赋值给了PCIHost的config寄存器`config_reg`。


### 指令执行

指令执行阶段会解析`config_reg`中保存的指令。

```c
memory_region_write_accessor -> pci_host_data_write -> pci_data_write

static void pci_host_data_write(void *opaque, hwaddr addr,
                                uint64_t val, unsigned len)
{
    PCIHostState *s = opaque;

    if (s->config_reg & (1u << 31))
        pci_data_write(s->bus, s->config_reg | (addr & 3), val, len);
}

void pci_data_write(PCIBus *s, uint32_t addr, uint32_t val, unsigned len)
{
    PCIDevice *pci_dev = pci_dev_find_by_addr(s, addr);
    uint32_t config_addr = addr & (PCI_CONFIG_SPACE_SIZE - 1);

    if (!pci_dev) {
        return;
    }

    pci_host_config_write_common(pci_dev, config_addr, PCI_CONFIG_SPACE_SIZE,
                                 val, len);
}
```

先是在`pci_dev_find_by_addr`中利用`config_reg`找到的目标pci设备，根据注释中指出的pci规范进行解析，从而能够找到目标设备。

```c
/*
 * PCI address
 * bit 16 - 24: bus number
 * bit  8 - 15: devfun number
 * bit  0 -  7: offset in configuration space of a given pci device
 */

/* the helper function to get a PCIDevice* for a given pci address */
static inline PCIDevice *pci_dev_find_by_addr(PCIBus *bus, uint32_t addr)
{
    uint8_t bus_num = addr >> 16;
    uint8_t devfn = addr >> 8;

    return pci_find_device(bus, bus_num, devfn);
}

PCIDevice *pci_find_device(PCIBus *bus, int bus_num, uint8_t devfn)
{
    bus = pci_find_bus_nr(bus, bus_num);

    if (!bus)
        return NULL;

    return bus->devices[devfn];
}
```

找到目标设备后在`pci_host_config_write_common`中利用`config_reg`找到目标设备config空间：

先是取出要操作的地址(即操作目标设备config空间的那个字段)。然后对该空间进行读写`pci_host_config_write_common`，其中会根据PCI设备类型执行相应回调函数完成操作。

```c
uint32_t config_addr = addr & (PCI_CONFIG_SPACE_SIZE - 1); 	// addr & 0x11

pci_host_config_write_common(pci_dev, config_addr, PCI_CONFIG_SPACE_SIZE,
                                 val, len);
```


## 读写设备内存空间

为了能够寻址到pci设备，会在设备config空间中记录该设备内存映射相关的信息。这就是config空间中的BAR(Base Address Register)的功能。

<img src="./pci_config_space.png" alt="" width="100%">

如图所示，BAR(Base Address Register)是PCIconfig空间中从0x10到0x24的6个register，用来定义PCI需要的配置空间大小以及**配置PCI设备占用的地址空间**。

BAR根据mmio和pio有两种不同的布局，其中bit0用于指示是内存是mmio还是pio，详见下图：

<img src="./bar_layout.png" alt="" width="100%">

每个PCI设备在BAR中描述自己需要占用多少地址空间，bios会通过pci枚举探测pci设备，然后读取其BAR进行合理的地址空间分配。这个过程读应QEMU中虚拟机reset阶段执行的`pci_update_mapping`，后文将会说明这点。

pci设备的BAR设置可以使用`pci_register_bar`完成，其中`io_regions`数组表示设备的BARs，数组中的每一项表示BARs中的一个寄存器，`pci_register_bar`完成一项BAR的设置(`pci_dev->io_regions[region_num]`)。

```c
void pci_register_bar(PCIDevice *pci_dev, int region_num,
                      uint8_t type, MemoryRegion *memory)
{
    PCIIORegion *r;
    uint32_t addr; /* offset in pci config space */
    uint64_t wmask;
    pcibus_t size = memory_region_size(memory);

    /* ... */

    r = &pci_dev->io_regions[region_num];
    r->addr = PCI_BAR_UNMAPPED;
    r->size = size;
    r->type = type;
    r->memory = memory;
    r->address_space = type & PCI_BASE_ADDRESS_SPACE_IO
                        ? pci_dev->bus->address_space_io
                        : pci_dev->bus->address_space_mem;
	/* ... */
}
```

以e1000网卡设备为例，其实例化函数`pci_e1000_realize`中设置了BAR0和BAR1，并在BAR中记录地址与MR绑定，最后虚拟机reset阶段统一将所有设备的所以BAR映射到内存中。

```c
pci_register_bar(pci_dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &d->mmio);
pci_register_bar(pci_dev, 1, PCI_BASE_ADDRESS_SPACE_IO, &d->io);
```

`pci_register_bar`仅是设置了BAR的的内容，真正根据BAR信息完成pci设备内存映射的是`pci_update_mapping`。

虚拟设备创建完成后进入虚拟机reset阶段`qemu_system_reset`，这个阶段会遍历所有的pci设备和所有的`io_regions`然后调用`pci_update_mapping`，根据之前`pci_register_bar`中设置的`addr`和对应的`memory`，为各个pci设备映射内存空间。

```c
static void pci_update_mappings(PCIDevice *d)
{
...
    for(i = 0; i < PCI_NUM_REGIONS; i++) {
        r = &d->io_regions[i];

        /* this region isn't registered */
        if (!r->size)
            continue;

		//get the address info stored in specific bar
        new_addr = pci_bar_address(d, i, r->type, r->size);

        /* This bar isn't changed */
        if (new_addr == r->addr)
            continue;

        /* now do the real mapping */
        if (r->addr != PCI_BAR_UNMAPPED) {
            memory_region_del_subregion(r->address_space, r->memory);
        }
        r->addr = new_addr;
        if (r->addr != PCI_BAR_UNMAPPED) {
            memory_region_add_subregion_overlap(r->address_space,
                                                r->addr, r->memory, 1);
        }                                       
    }   
    pci_update_vga(d);
}
```

映射完成后就能通过mmio和pio的方式访问pci设备了??

Q??，映射后就是不是就直接访存了?如cpu直接访存mm，copy数据到VGA的mmio就能显示到屏幕了?

Q?? config space和设备读写的关系如何?该了config空间的某一点后就能能够命令设备读写?那这个过程是怎样的??

## 基础设施

这里的基础设施指的是PCI相关结构体和API。如pcihost要找到pci设备需要哪些结构，pci设备config空间该如何维护等。

pcibus维护一个pci设备列表`PCIDevice bus->devices[256]`记录挂在pci总线上的设备，pcihost可以根据pcibus协议解析cpu传入的addr获得pci设备的索引，从而找到目标pci设备。

```c
struct PCIBus {
    BusState qbus;
	/* ... */
    PCIDevice *devices[PCI_SLOT_MAX * PCI_FUNC_MAX];
    PCIDevice *parent_dev;
    MemoryRegion *address_space_mem;
    MemoryRegion *address_space_io;
	/* ... */
};
```

pci设备的config空间由pci设备自己维护(PCIDevice结构的`uint8_t *config`成员)。设备实例化时会初始化这段空间(`uint8_t *config`，256byte的config空间)，并挂载上总线`bus->devices[devfn] = pci_dev`。之后的设备内存空间映射和config空间读写都依赖于`*config`。

```c
struct PCIDevice {
    DeviceState qdev;
    bool partially_hotplugged;

    /* PCI config space */
    uint8_t *config;
	/* ... */
}
```

<img src="./pci_config_space.png" alt="" width="100%">

pci协议规范规定了config空间中各个bit的含义，在`hw/pci/pci.c`中提供了许多API、`pci_regs.h`中定义了许多config空间相关的宏，方便我们对config空间进行操作。如`pci_config_set_vendor_id`API可以完成config空间vendor id段的填写：

```c
static inline void
pci_config_set_vendor_id(uint8_t *pci_config, uint16_t val)
{
    pci_set_word(&pci_config[PCI_VENDOR_ID], val);
}
```

`do_pci_register_device`可以方便的配置一个PCI设备。

```
static PCIDevice *do_pci_register_device(PCIDevice *pci_dev,
                                         const char *name, int devfn,
                                         Error **errp)
{
    PCIConfigReadFunc *config_read = pc->config_read;
    PCIConfigWriteFunc *config_write = pc->config_write;

	/* ... */
	memory_region_init(&pci_dev->bus_master_container_region, OBJECT(pci_dev),
                       "bus master container", UINT64_MAX);
    address_space_init(&pci_dev->bus_master_as,
                       &pci_dev->bus_master_container_region, pci_dev->name);

    pci_config_alloc(pci_dev);

    pci_config_set_vendor_id(pci_dev->config, pc->vendor_id);
    pci_config_set_device_id(pci_dev->config, pc->device_id);
    pci_config_set_revision(pci_dev->config, pc->revision);
    pci_config_set_class(pci_dev->config, pc->class_id);
	/* ... */
	bus->devices[devfn] = pci_dev;
	/* ... */
}
```


