---
title: QEMU总线模拟与设备通信
date: 2021-08-31
tags: 
- qemu
mathjax: true
---

# Preface

现实世界中设备之间通过总线互联/通信，其中sysbus一般负责与cpu直接相连的设备与cpu的通信，外围设备则会通过各种类型的总线(如pci总线)间接地与cpu通信。本文主要讨论QEMU中是如何对总线进行模拟来实现设备间通信。

QEMU中实现了`BusState`和`SysBusDevice`等BUS类来表示总线类型的继承关系，将设备强制转换成`DeviceState`结构后使用`DeviceState`结构的`parent_bus`和`child_bus`成员配合上bus相关类型转换的宏(如`BUS()`, `PCI_BUS()`)即可获得所连接的bus，从而可以联系到连接在总线的其他设备(见实例1)。但并不是所有通信都会用的这些结构，还可以使用设备结构体中的记录的成员直接通信(如中断传递，见实例2和3)。

下面通过三个实例来说明QEMU设备间通信的方式。


# 通信方式

QEMU中有多种方式实现设备与设备的互联，可以通过结构体中的成员互联、可以通过类的继承关系互联也可以通过QOM的property机制进行互联，从而模拟现实中设备通过总线进行互联和通信。


## 实例1：使用DeviceState中的层次关系

所有设备都是`DeviceState`的子类，`DeviceState`中又记录连接上级的`parent_bus`和连接下级的`child_bus`。设备的初始化一般通过`sysbus_realize_and_unref()- > qdev_realize_and_unref -> qdev_realize -> qdev_set_parent_bus`完成。`qdev_set_parent_bus`中就会为`DeviceState`的`parent_bus`成员赋值。`sysbus_realize_and_unref()`的情况就是`dev->parent_bus = sysbus_get_default()`让`parent_bus`指向系统总线。

```
struct DeviceState {
    /* ... */
    BusState *parent_bus;
    QLIST_HEAD(, NamedGPIOList) gpios;
    QLIST_HEAD(, BusState) child_bus;
    /* ... */
};
```

因此一种类型的设备可以通过类型转换成`DeviceState`，再利用`DeviceState`联系到上级或下级设备。

这里以PCI设备向上设备(南桥芯片piix3)传递中断信号为例：

```
gdb	 \
  -ex "b piix3_set_irq" \
  --args \
qemu-system-x86_64 \
  -machine kernel-irqchip=off \
  -m 8G \
  -kernel ./vmlinux \
```

核心调用流程如下：

```
pci_change_irq_level -> pci_bus_change_irq_level -> piix3_set_irq
```

首先在`pci_change_irq_level`中通过`pci_get_bus(pci_dev)`获取父PCI总线。`pci_get_bus()`展开为`PCI_BUS(qdev_get_parent_bus(DEVICE(dev)))`，本质上就是返回了`DeviceState`的`parent_bus`成员。又因为`TYPE_PCI`继承了`TYPE_BUS`从而可以通过`PCI_BUS`宏进行类型转换成对应的PCI总线，进而联系到对应的PCI设备

```
static const TypeInfo pci_bus_info = {
    .name = TYPE_PCI_BUS,
    .parent = TYPE_BUS,
    /* ... */
};

static void pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
{
    PCIBus *bus;
    for (;;) {
        bus = pci_get_bus(pci_dev); 	// <========== get bus
        irq_num = bus->map_irq(pci_dev, irq_num);
        if (bus->set_irq)
            break;
        pci_dev = bus->parent_dev;
    }
    pci_bus_change_irq_level(bus, irq_num, change);
}

pci_bus_change_irq_level(){
	bus->set_irq(bus->irq_opaque, irq_num, bus->irq_count[irq_num] != 0);
}
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_device_communication/pci_to_piix.png" alt="" width="100%">


## 实例2：设备结构体自己维护bus成员

像`PCIHost`这样的总线设备其本身的创建就是为了模拟PCI总线(或者说PCI总线控制器)，所以其结构中要记录维护的bus信息，以`PCIHostState`为例，其`bus`成员就是对连接其他PCI设备的关键。

```
struct PCIHostState {
    SysBusDevice busdev;

    MemoryRegion conf_mem;
    MemoryRegion data_mem;
    MemoryRegion mmcfg;
    uint32_t config_reg;
    bool mig_enabled;
    PCIBus *bus; 			// <====

    QLIST_ENTRY(PCIHostState) next;
};
```

cpu执行指令，翻译得到PCIHost设备绑定的`MemoryRegion`，触发设备的模拟/读写：

```
memory_region_write_accessor -> pci_host_data_write
```

这里将`MemoryRegion`绑定的PCIHost`opaque`传入，再通过`PCIHostState`的`bus`成员和`addr`信息联系其他设备，进行下一步操作。

```
static void pci_host_data_write(void *opaque, hwaddr addr,
                                uint64_t val, unsigned len)
{
    PCIHostState *s = opaque;

    if (s->config_reg & (1u << 31))
        pci_data_write(s->bus, s->config_reg | (addr & 3), val, len);
}

void pci_data_write(PCIBus *s, uint32_t addr, uint32_t val, unsigned len)
{
    PCIDevice *pci_dev = pci_dev_find_by_addr(s, addr); 			// <========= 找到目标设备
    uint32_t config_addr = addr & (PCI_CONFIG_SPACE_SIZE - 1);

    if (!pci_dev) {
        return;
    }

    pci_host_config_write_common(pci_dev, config_addr, PCI_CONFIG_SPACE_SIZE,
                                 val, len);
}
```

根据`addr`找到目标PCI设备后在调用目标设备回调函数：

```
pci_host_config_write_common(pci_dev, ...) -> pci_dev->config_write() -> i440fx_write_config
```


## 实例3：使用property建立连接

上面两例子说明了QOM可以使用继承和维护结构体成员的方式联系其他设备。

而设备连接关系的建立可以借助QOM的`property`机制完成。如中断控制器要向cpu传递中断就要调用对应cpu的`irq_handler`，那pic结构体中的中断线成员就是pic与cpu连接的关键。通过下面步骤就能将cpu的`irq_handler`记录到pic的结构体中，从而完成连接的建立：

- pic初始化，将结构体中代表中断线的成员添加到`property`表
	* 简单地说property是每个设备都有的一个哈希表，用于记录设备的属性和连接关系等，方便需要时获取
- cpu初始化后通过`property`表找到中断控制器设备的中断线，将该cpu的handler赋值给它

这样一来pic需要向cpu传递中断时，通过自身结构的中断线成员就能调用到cpu的`irq_handler`从而实现中断传递的效果。

下面以`openpic`中断控制器与`PowerPC e500`cpu连接的建立为例说明这一点：

首先`qdev_new(TYPE_OPENPIC)`创建一个openpic设备，在其实例化函数`openpic_realize`中使用`sysbus_init_irq -> qdev_init_gpio_out_named -> object_property_add_link`创建了"sysbus-irq"属性，相当于对外暴露了中断线：

```
static void openpic_realize(DeviceState *dev, Error **errp)
{
    OpenPICState *opp = OPENPIC(dev);
	/* ... */
	for (i = 0; i < opp->nb_cpus; i++) {
        opp->dst[i].irqs = g_new0(qemu_irq, OPENPIC_OUTPUT_NB);
        for (j = 0; j < OPENPIC_OUTPUT_NB; j++) {
            sysbus_init_irq(d, &opp->dst[i].irqs[j]);
        }
		/* ... */
    }
	/* ... */
}
```

openpic使用`OpenPICState->dst[i].irqs`维护连接到cpu的中断线。`object_property_add_link`创建了一个"sysbus-irq"到`OpenPICState->dst[i].irqs`指针的键值对。之后在cpu初始化的过程中调用`sysbus_connect_irq -> qdev_connect_gpio_out_named -> ... -> object_set_link_property`通过"sysbus-irq"找到pic的irq指针，从而实现"将cpu的中断线赋值给openpic的irq指针，连接pic到cpu的连接"。

```
struct OpenPICState {
	/* ... */
    IRQDest dst[MAX_CPU];
	/* ... */
};

typedef struct IRQDest {
	/* ... */
    qemu_irq *irqs;
	/* ... */
} IRQDest;

// connect
static void object_set_link_property(Object *obj, Visitor *v,
                                     const char *name, void *opaque,
                                     Error **errp)
{
	/* ... */
    Object **targetp = object_link_get_targetp(obj, prop); 	  // 获取property中保存的irq执行
	/* ... */
	new_target = object_resolve_link(obj, name, path, errp);  // 获取传入的cpu中断线qemu_irq
	/* ... */
    *targetp = new_target; 									  // irq指针指向cpu中断线qemu_irq
	/* ... */
}
```

这样当openpic需要与cpu联系时可以通过调用`opp->dst[i]->irqs->handler()`实现。


# 中断传递

现实中，中断会通过总线层层传递。下面通过QEMU中一个中断传递的例子体会设备通信(或中断传递)的过程。

```
gdb	 \
  -ex "b serial_update_irq" \
  --args \
qemu-system-x86_64 \
  -machine kernel-irqchip=off \
  -m 8G \
  -kernel ./vmlinux \
```

首先cpu执行一条制定，通过`memory_region_write_accessor()`翻译到`MemoryRegion`，从而找到与其绑定的设备`serial-mm`。

由于`serial-mm`设备的`SerialState`结构的`irq`成员记录了所连接的中断线信息，而`irq`成员对应的`IRQState`结构的`opaque`成员又记录了所连接到的设备，所以之后调用`serial_update_irq`中断就会一层一层的在设备结构体间传递，如下所示：

```
struct IRQState {
    Object parent_obj;
    qemu_irq_handler handler;
    void *opaque;
    int n;
};

Dev1.irq.opaque -> Dev2 -> Dev2.irq.opaque -> Dev3 -> Dev3.irq.handler();
```

这里serial设备的传递过程大体如下：

- 1. serial设备根据它的`irq`成员传递到`GSI`设备(Global System Interrupts)。
- 2. `irq->handler`进入`GSI`设备后继续调用`qemu_set_irq`传递，这次是传递`GSI`设备的`irq`
- 3. 从`GSI`的irq进入到`pic_set_irq()`
- 4. pic继续同样的方法传递，知道传递到cpu，修改标记位，cpu循环退出，从而模拟一次中断

```
static void serial_update_irq(SerialState *s)
{
	/* ... */
	qemu_irq_raise(s->irq);
	/* ... */
}

qemu_irq_raise() -> qemu_set_irq()

void qemu_set_irq(qemu_irq irq, int level)
{
    irq->handler(irq->opaque, irq->n, level);  		// gsi_handler
}

void gsi_handler(void *opaque, int n, int level)
{
    GSIState *s = opaque;
	/* ... */
    qemu_set_irq(s->ioapic_irq[n], level);
}

qemu_irq_raise() -> qemu_set_irq()

static void pic_set_irq(void *opaque, int irq, int level)
{
	PICCommonState *s = opaque;
	/* ... */
	pic_update_irq(s);
}

pic_update_irq -> qemu_irq_raise -> pic_irq_request
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_device_communication/int_deliver.png" alt="" width="100%">

