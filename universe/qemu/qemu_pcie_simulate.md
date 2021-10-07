---
title: qemu中对pcie的模拟
date: 2021-10-04
tags: 
- qemu
- pcie
mathjax: true
---

# Overview

pcie对pci的功能进行了扩展，同步保留的对pci总线规范的兼容。这里仅对以下几点进行说明：

- root complex
	* root complex是个抽象的概念，并不是某个具体的器件，表示一组pcie系统元素(桥、交换机、端口等)。简单的讲RC就是cpu与pcie拓扑结构的连接点(root)，而这个连接点可以包含很多组件(所以叫complex)。主要功能(与pcie设备通信)还是通过host bridge实现
	* ref:
		+ Terms and Acronyms -> Root Complex
		+ Terms and Acronyms -> Host Bridge
		+ 1.3.1. Root Complex
		+ 7.2.2.1. Host Bridge Requirements
		```
		Root complex
		 
		A defined System Element that includes zero or more Host Bridges, zero or more
		Root Complex Integrated Endpoints, zero or more Root Complex Event
		Collectors, and one or more Root Ports.
		```
	* qemu: `qemu/hw/pci/pcie_host.c: pcie_host_init`
- configuration space
	* pcie为了兼容pci，在支持mmio的同时也支持pio的访问方式。mmio的访问方式是为使用pcie扩展功能设置的，遵循ECAM定义的规则。
	* ref:
		+ *7.2. PCI Express Configuration Mechanisms*
		+ *7.2.2. PCI Express Enhanced Configuration Access Mechanism (ECAM)*
		+ 使用ECAM可以进行pcie扩展的操作：*7.8. PCI Express Capability Structure*，*7.9. PCI Express Extended Capabilities*
	* qemu: 
		+ `qemu/hw/pci/pcie_host.c`，提供了mmio支持，对pio的支持可以通过继承`TYPE_PCIE_HOST_BRIDGE`类后独立实现
		+ `qemu/hw/pci/pcie_host.h`，提供了ECAM相关的宏
- interrupt
	* pcie保留了对pci中断的支持(interrup line和interrup pin寄存器)，但中断信号的触发不同于pci中使用4个中断引脚，而是使用*in-band Messages*。但在控制器看了无非就是`Assert_INTx and Deassert_INTx`，所以qemu模拟仍然是经过`pci_irq_handler`
	* ref:
		+ *6.1.2. PCI Compatible INTx Emulation*
		+ *6.1.3. INTx Emulation Software Model*
- TODO: MSI，也是一种中断方式，可以做但感觉不是主线


# configuration space

## pci兼容性

> 7.2. PCI Express Configuration Mechanisms
>
> The PCI 3.0 compatible Configuration Space can be accessed using either the
> mechanism defined in the PCI Local Bus Specification or the PCI Express Enhanced Configuration
> Access Mechanism (ECAM) described later in this section. Accesses made using either access mechanism are equivalent.

为了能够兼容pci，pcie中configuration space的前256byte是与pci规范中的规定是兼容的。并且因为pci的通过pio访问configuration space的，所以pcie中也要支持通过pio来访问configuration space。所以pcie中要提供mmio访问config space的接口也要提供通过pio访问config space的接口。

在qemu中的模拟，以`q35`为例：

- `qemu/hw/pci/pcie_host.c`中定义了pcie host类，它仅实现了pcie中mmio访问config space部分
- `qemu/hw/pci-host/q35.c`通过继承`TYPE_PCIE_HOST_BRIDGE`(即继承`pcie_host.c`中mmio接口的实现)，再自己实现pio，即可实现对mmio和pio的支持

```c
// pcie_host.c
void pcie_host_mmcfg_map(PCIExpressHost *e, hwaddr addr,
                         uint32_t size)
{
    pcie_host_mmcfg_init(e, size);
    e->base_addr = addr;
    memory_region_add_subregion(get_system_memory(), e->base_addr, &e->mmio);
}

// q35.c
static void q35_host_realize(DeviceState *dev, Error **errp)
{
	...
    sysbus_add_io(sbd, MCH_HOST_BRIDGE_CONFIG_ADDR, &pci->conf_mem);
    sysbus_init_ioports(sbd, MCH_HOST_BRIDGE_CONFIG_ADDR, 4);

    sysbus_add_io(sbd, MCH_HOST_BRIDGE_CONFIG_DATA, &pci->data_mem);
    sysbus_init_ioports(sbd, MCH_HOST_BRIDGE_CONFIG_DATA, 4);
	...
}
```

这样访问pcie设备既可以通过mmio完成，也可以通过pio完成。


## pcie扩展功能与ECAM

> 7.2.2. PCI Express Enhanced Configuration Access Mechanism (ECAM)

pcie是configuration space有4KB，其中前256byte是兼容pci的，其余部分将是pcie特有的，必须通过mmio，使用PCI Express Enhanced Configuration Access Mechanism (ECAM)的方式访问。

通过mmio访问pcie，指令格式遵从以下规则：

- bit 29 - 64: mmio base address
- bit 20 - 28: bus number
- bit 15 - 19: device number
- bit 12 - 14: function number
- bit  0 - 11: offset in configuration space of a given device

qemu中`pcie.c`中对pcie功能进行了模拟。下面以`qemu/hw/pci/pcie_host.c:pcie_mmcfg_data_read`为例，说明pcie中configuration space的访问流程。

```c
static uint64_t pcie_mmcfg_data_read(void *opaque,
                                     hwaddr mmcfg_addr,
                                     unsigned len)
{
    PCIExpressHost *e = opaque;
    PCIBus *s = e->pci.bus;
    PCIDevice *pci_dev = pcie_dev_find_by_mmcfg_addr(s, mmcfg_addr);
    uint32_t addr;
    uint32_t limit;

    if (!pci_dev) {
        return ~0x0;
    }
    addr = PCIE_MMCFG_CONFOFFSET(mmcfg_addr);
    limit = pci_config_size(pci_dev);
    return pci_host_config_read_common(pci_dev, addr, limit, len);
}
```

根据ECAM解析出bus number(bit 20-28)，device number(bit 15-29)和要访问的寄存器(bit 0-11)，然后就能复用pci中的访问流程了：

```c
#define PCIE_MMCFG_SIZE_MAX             (1ULL << 29)
#define PCIE_MMCFG_SIZE_MIN             (1ULL << 20)
#define PCIE_MMCFG_BUS_BIT              20
#define PCIE_MMCFG_BUS_MASK             0x1ff
#define PCIE_MMCFG_DEVFN_BIT            12
#define PCIE_MMCFG_DEVFN_MASK           0xff
#define PCIE_MMCFG_CONFOFFSET_MASK      0xfff
#define PCIE_MMCFG_BUS(addr)            (((addr) >> PCIE_MMCFG_BUS_BIT) & \
                                         PCIE_MMCFG_BUS_MASK)
#define PCIE_MMCFG_DEVFN(addr)          (((addr) >> PCIE_MMCFG_DEVFN_BIT) & \
                                         PCIE_MMCFG_DEVFN_MASK)
#define PCIE_MMCFG_CONFOFFSET(addr)     ((addr) & PCIE_MMCFG_CONFOFFSET_MASK)
```

1. 找总线，找设备
	```
	pcie_dev_find_by_mmcfg_addr -> pci_find_device

	static inline PCIDevice *pcie_dev_find_by_mmcfg_addr(PCIBus *s,
														 uint32_t mmcfg_addr)
	{
		return pci_find_device(s, PCIE_MMCFG_BUS(mmcfg_addr),
							   PCIE_MMCFG_DEVFN(mmcfg_addr));
	}
	```
2. 找要访问的寄存器
	```c
	addr = PCIE_MMCFG_CONFOFFSET(mmcfg_addr);
	```
3. 复用pci模拟中的访问方式
	```
	pci_host_config_read_common(pci_dev, addr, limit, len);
	```


# pcie中断支持

不同于pci中有4个独立的中断引脚，pcie中使用*in-band Messages*传递中断触发信息(two type message: `Assert_INTx and Deassert_INTx`)。并且要求所以要产生中断的pcie设备都要支持MSI的中断方式。

> All PCI Express device Functions that are capable of generating interrupts must support MSI or MSI-X or both.


