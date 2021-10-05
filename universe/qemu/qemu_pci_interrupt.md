---
title: pci对中断的支持
date: 2021-10-05
tags: 
- qemu
mathjax: true
---


# Preface

> PCI Local Bus Specification: 2.2.6. Interrupt Pins (Optional)

pci总线规范为pci设备准备了4个中断引脚(INTx#)用于传递中断。设备所使用的中断引脚信息保存在configuration space的*Interrupt Pin Register*中。这4个中断引脚在设备间的共享的，引脚的分配pci spec推荐遵循一定的规则，详见`Implementation Note: Interrupt Routing`。*Interrup Line Register*中记录设备对应的IRQ，可以使用pci总线规范中推荐的规则建立设备`INTx#`到`IRQ`的联系，见`2.2.6. Interrupt Pins (Optional): Interrupt Routing`。


# 中断支持

- 连接的中断线
	* `Interrupt Pin Register`中记录设备所连接的中断引脚(`INTA#, INTB#, INTC#, INTD#`)
	* ref: 
		+ *PCI Local Bus Specification: 2.2.6. Interrupt Pins (Optional)*
		+ *PCI Local Bus Specification: 6.2.4. Miscellaneous Registers*
	* qemu: `qemu/hw/pci/pci.c: pci_intx()`
- 中断处理函数
	* `Interrupt Line Register`中记录设备所需的中断向量
	* ref: 
		+ *PCI Local Bus Specification: 6.2.4. Miscellaneous Registers*
	* qemu: 似乎是根据routing规则算出irq，而不是读取`Interrup Line Register`。如`qemu/hw/isa/piix3.c: pci_slot_get_pirq()`注释写到：`Return the global irq number corresponding to a given device irq pin.`
- 中断状态
	* 中断禁止：`Command Register: bit 10`
	* 中断预备：`Device Status Register: bit 3`
	* ref:
		+ *PCI LOCAL BUS SPECIFICATION, REV. 3.0: Device Control*
		+ *PCI LOCAL BUS SPECIFICATION, REV. 3.0: Device Status*
	* qemu: 
		+ `qemu/hw/pci/pci.c: pci_update_irq_status()`
		+ `qemu/hw/pci/pci.c: pci_irq_disabled()`


*PCI-to-PCI Bridge Architecture Specification: 11.2. System Initialization*介绍了系统初始化的流程，其中就包括中断的初始化：*11.2.3. Writing IRQ Numbers into Interrupt Line Register(s)*。

简单的说就是：系统初始化程序(如BIOS)会感知到pci设备的连接状况，然后将设备的`INTx#`与`IRQ`绑定(Interrup Routing)，分别填入configuration space的`Interrupt Pin Register`和`Interrupt Line Register`中。中断触发时将会通过`Interrupt Line Register`中的内容寻找中断例程。

pci-pci bridge spec中(*PCI-to-PCI Bridge Architecture Specification: Chapter 9 Interrupt Support*)对中断的支持进行了较为详细的说明。

qemu中的模拟：`qemu/hw/pci/pci.c`，约1435行到1525行提供了整套pci中断处理流程的模拟，大体过程如下：

1. 设备初始化时，如果需要使用pci提供的中断支持，使用`pci_allocate_irq`申请pci中断，这样该设备中断触发时将交由`pci_irq_handler`处理
	```c
	pci_ich9_ahci_realize
		d->ahci.irq = pci_allocate_irq(dev);


	qemu_irq pci_allocate_irq(PCIDevice *pci_dev)
	{
		int intx = pci_intx(pci_dev);

		return qemu_allocate_irq(pci_irq_handler, pci_dev, intx);
	}
	```
2. `pci_irq_handler`通过`Interrup Pin Register`获取设备的`INTx`信息：`pci_intx(pci_dev)`。通过`Status`寄存器更新中断状态等
	```
	static void pci_irq_handler(void *opaque, int irq_num, int level)
	{
		PCIDevice *pci_dev = opaque;
		int change;

		change = level - pci_irq_state(pci_dev, irq_num);
		if (!change)
			return;

		pci_set_irq_state(pci_dev, irq_num, level);
		pci_update_irq_status(pci_dev);
		if (pci_irq_disabled(pci_dev))
			return;
		pci_change_irq_level(pci_dev, irq_num, change);
	}
	```
3. 获取到intx后在`pci_change_irq_level`中完成irq查询和中断触发等操作。使用`bus->map_irq`回调函数查找设备对应的IRQ
	```c
	static void pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
	{
		PCIBus *bus;
		for (;;) {
			bus = pci_get_bus(pci_dev);
			irq_num = bus->map_irq(pci_dev, irq_num);
			if (bus->set_irq)
				break;
			pci_dev = bus->parent_dev;
		}
		pci_bus_change_irq_level(bus, irq_num, change);
	}
	```
4. 最后在`pci_bus_change_irq_level`中调用`bus->set_irq()`根据IRQ触发中断，层层传递至cpu
	```c
	static void pci_bus_change_irq_level(PCIBus *bus, int irq_num, int change)
	{
		assert(irq_num >= 0);
		assert(irq_num < bus->nirq);
		bus->irq_count[irq_num] += change;
		bus->set_irq(bus->irq_opaque, irq_num, bus->irq_count[irq_num] != 0);
	}
	```

`bus->map_irq`和`bus->set_irq`是在控制器设备初始化时设置(`pci_bus_irqs()`)的，表示pci中断线所连接到的控制器，再由控制器处理然后传递中断。

```c
static void pc_q35_init(MachineState *machine)
{
	...
    pci_bus_irqs(host_bus, ich9_lpc_set_irq, ich9_lpc_map_irq, ich9_lpc,
                 ICH9_LPC_NB_PIRQS);
	...
}

void pci_bus_irqs(PCIBus *bus, pci_set_irq_fn set_irq, pci_map_irq_fn map_irq,
                  void *irq_opaque, int nirq)
{
    bus->set_irq = set_irq;
    bus->map_irq = map_irq;
    bus->irq_opaque = irq_opaque;
    bus->nirq = nirq;
    bus->irq_count = g_malloc0(nirq * sizeof(bus->irq_count[0]));
}
```

# TODO

- MSI
- Q
	* 一个设备初始化后irq就确定了就不改变了?
