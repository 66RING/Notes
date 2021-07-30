---
title: qemu架构中创建内存的方法
date: 2021-05-24
tags: 
- qemu
mathjax: true
---

# 引言

一种类型的虚拟机有其特定的初始化函数以完成虚拟机内存初始化等操作。调用哪个初始化函数在构建虚拟机类时指定：

```c
static const TypeInfo my_machine_info = {
    ...
    .class_init    = my_machine_class_init,
};

static void my_machine_registor(void)
{
    type_register_static(&my_machine_info);
}
type_init(my_machine_registor);
```

其中`my_machine_class_init`是自定义的机器类的构造方法，用于初始化虚拟机类的各种信息，其中就包括指定虚拟机初始化主函数，如：

```c
static void my_machine_class_init(ObjectClass *oc, void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);
    mc->init = my_machine_init;
	...
}
```

使用`MACHINE_CLASS(oc)`宏将`ObjectClass`类型强制转换成`MachineClass mc`对象，然后让`mc->init`赋值为自定义的虚拟机初始化主函数`my_machine_init`。这样qemu框架在创建虚拟机时就能使用我们指定的方法了。

通过上面的过程我们知道，令`mc->init = my_machine_init`后qemu就会执行我们定义的虚拟机初始化函数`my_machine_init`。虚拟机初始化包括虚拟机内存的创建，因此我们要在`my_machine_init`函数中完成内存初始化部分操作。

我要独立设计内存创建方法则要在`my_machine_init`函数中完成内存初始化部分的操作。如初始化ram、rom等

虚拟机初始化函数会接受一个`MachineState *machine`作为参数，其中包括虚拟机的基本信息，这些信息由qemu解析命令行参数获得，其中就包含管理虚拟机ram的MemoryRegion`machine->ram`和ram大小的信息`machine->ram_size`等。

下面将分别介绍x86、arm和powerpc架构虚拟机创建内存的方法。


# x86

x86机器为了实现向后兼容，需要用一个MemoryRegion代表低地址空间的内存，用另一个MemoryRegion代表地址空间的内存。将代表低地址空间的mr称为`ram_below_4g`，代表高地址空间的mr称为`ram_above_4g`。虽然名称上是以4g为分界点，但实际中是以3g为分界点，即分配给虚拟机的内存大于3g就需要创建`ram_above_4g`了。

通过`machine->ram_size`可知道为虚拟机分配了多少内存，如果`ram_size`大于4g，就需要创建一个mr(`ram_above_4g`)来管理高内存地址空间。因此我们的内存初始化需要进行如下操作：

```c
static void my_machine_init(MachineState *machine)
{
	ram_addr_t lowmem = 0xc0000000; /* 3G */
	if (machine->ram_size > lowmem) {
        above_4g_mem_size = machine->ram_size - lowmem;
        below_4g_mem_size = lowmem;
    } else {
        above_4g_mem_size = 0;
        below_4g_mem_size = machine->ram_size;
    }
	...
	MemoryRegion *ram_below_4g = g_malloc(sizeof(*ram_below_4g));
	memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", machine->ram, // 创建一个表示低内存地址空间的mr
							 0, below_4g_mem_size);
	memory_region_add_subregion(system_memory, 0, ram_below_4g);
	...
	if (machine->ram_size > lowmem) {

		MemoryRegion *ram_above_4g = g_malloc(sizeof(*ram_above_4g));
		memory_region_init_alias(ram_above_4g, NULL, "ram-above-4g",  // 创建一个表示高内存地址空间的mr
								 machine->ram,
								 below_4g_mem_size,
								 above_4g_mem_size);
		memory_region_add_subregion(system_memory, 0x100000000ULL, ram_above_4g);
	}
	...
	x86_bios_rom_init(machine, "bios.bin", rom_memory, true);
}
```

首先根据`machine->ram_size`提供的内存信息进行预处理。如果分配给虚拟机的内存大于规定的阀值`lowmem`(3G)，那么就要创建代表高地址的mr`ram_above_4g`，其大小就是低3G以外的部分`above_4g_mem_size = machine->ram_size - lowmem`

如果给虚拟机的内存低于阀值3G则仅创建代表低地址空间的mr，其大小`below_4g_mem_size = machine->ram_size`，见如下代码

```c
ram_addr_t lowmem = 0xc0000000; /* 3G */
if (machine->ram_size > lowmem) {
	above_4g_mem_size = machine->ram_size - lowmem;
	below_4g_mem_size = lowmem;
} else {
	above_4g_mem_size = 0;
	below_4g_mem_size = machine->ram_size;
}
```

无论是创建代表高地址mr还是创建代表低地址的mr，都要通过`memory_region_init_alias()`将其创建为`machine->ram`的别名。表示其管理`machine->ram`的一部分。

然后通过`memory_region_add_subregion()`接口函数指定高地址、低地址两部分mr在地址空间中的起始位置。

`memory_region_add_subregion()`的第一个参数为被加入到的MemoryRegion，第二个参数为待加入的mr在被加入mr中的位置，第三个参数为待加入的mr

代表低地址mr的起始位置为系统内存空间(`system_memory`)的0。代表高地址mr的起始位置为`system_memory`中的4G。实际代码示例如下：

```
MemoryRegion *ram_below_4g = g_malloc(sizeof(*ram_below_4g));
memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", machine->ram, // 创建一个表示低4g内存空间的mr
						 0, below_4g_mem_size);
memory_region_add_subregion(system_memory, 0, ram_below_4g);
```

如果虚拟机的内存大于4g，则还会创意一个管理高4g空间的MemoryRegion结构`ram_above_4g`，其同样是`machine->ram`的一个别名，其地址空间从4G开始。

```c
if (machine->ram_size > lowmem) {

	MemoryRegion *ram_above_4g = g_malloc(sizeof(*ram_above_4g));
	memory_region_init_alias(ram_above_4g, NULL, "ram-above-4g",  // 创建一个表示高4g内存空间的mr
							 machine->ram,
							 below_4g_mem_size,
							 above_4g_mem_size);
	memory_region_add_subregion(system_memory, 0x100000000ULL, ram_above_4g);
}
```

之后可以通过`x86_bios_rom_init(machine, "bios.bin", rom_memory, true)`函数接口创建`bios_rom`，其原型如下：

```c
void x86_bios_rom_init(MachineState *ms, const char *default_firmware,
                       MemoryRegion *rom_memory, bool isapc_ram_fw)
```

第一个参数为`machine`，第二个参数为默认设备名，第三个参数为用于管理的MemoryRegion，第四个参数为标记


# arm

arm架构虚拟机的内存创建同理，都是要在虚拟机初始化主函数中完成，设自定义的虚拟机初始化函数为`my_machine_init`，则`my_machine_init`大致应包含如下内容：

```c
static void my_machine_init(MachineState *machine)
{
    VirtMachineState *vms = VIRT_MACHINE(machine);
	if (!vms->memmap) {
        virt_set_memmap(vms);
    }
	...
	memory_region_add_subregion(sysmem, vms->memmap[VIRT_MEM].base,
							machine->ram);
	...
    create_gic(vms);
	...
	create_pcie(vms);
	...
	virt_flash_fdt(vms, sysmem, secure_sysmem ?: sysmem);
	...
    fdt_add_pmu_nodes(vms);
	...
}
```

arm架构通过`vms`结构管理虚拟机的信息，其中就包括内存地址划分方法。通过`virt_set_memmap(vms)`接口函数就能完成vms结构的初始化工作。其中主要会赋值`vms->memmap`，这是一个字典结构，每一项都包含内存空间的起始地址和大小。

```c
static const MemMapEntry base_memmap[] = {
    /* Space up to 0x8000000 is reserved for a boot ROM */
    [VIRT_FLASH] =              {          0, 0x08000000 },
    [VIRT_CPUPERIPHS] =         { 0x08000000, 0x00020000 },
    /* GIC distributor and CPU interfaces sit inside the CPU peripheral space */
    [VIRT_GIC_DIST] =           { 0x08000000, 0x00010000 },
    [VIRT_GIC_CPU] =            { 0x08010000, 0x00010000 },
    [VIRT_GIC_V2M] =            { 0x08020000, 0x00001000 },
    [VIRT_GIC_HYP] =            { 0x08030000, 0x00010000 },
    [VIRT_GIC_VCPU] =           { 0x08040000, 0x00010000 },
    /* The space in between here is reserved for GICv3 CPU/vCPU/HYP */
    [VIRT_GIC_ITS] =            { 0x08080000, 0x00020000 },
    /* This redistributor space allows up to 2*64kB*123 CPUs */
    [VIRT_GIC_REDIST] =         { 0x080A0000, 0x00F60000 },
    [VIRT_UART] =               { 0x09000000, 0x00001000 },
    [VIRT_RTC] =                { 0x09010000, 0x00001000 },
    [VIRT_FW_CFG] =             { 0x09020000, 0x00000018 },
    [VIRT_GPIO] =               { 0x09030000, 0x00001000 },
    [VIRT_SECURE_UART] =        { 0x09040000, 0x00001000 },
    [VIRT_SMMU] =               { 0x09050000, 0x00020000 },
    [VIRT_PCDIMM_ACPI] =        { 0x09070000, MEMORY_HOTPLUG_IO_LEN },
    [VIRT_ACPI_GED] =           { 0x09080000, ACPI_GED_EVT_SEL_LEN },
    [VIRT_NVDIMM_ACPI] =        { 0x09090000, NVDIMM_ACPI_IO_LEN},
    [VIRT_PVTIME] =             { 0x090a0000, 0x00010000 },
    [VIRT_SECURE_GPIO] =        { 0x090b0000, 0x00001000 },
    [VIRT_MMIO] =               { 0x0a000000, 0x00000200 },
    /* ...repeating for a total of NUM_VIRTIO_TRANSPORTS, each of that size */
    [VIRT_PLATFORM_BUS] =       { 0x0c000000, 0x02000000 },
    [VIRT_SECURE_MEM] =         { 0x0e000000, 0x01000000 },
    [VIRT_PCIE_MMIO] =          { 0x10000000, 0x2eff0000 },
    [VIRT_PCIE_PIO] =           { 0x3eff0000, 0x00010000 },
    [VIRT_PCIE_ECAM] =          { 0x3f000000, 0x01000000 },
    /* Actual RAM size depends on initial RAM and device memory settings */
    [VIRT_MEM] =                { GiB, LEGACY_RAMLIMIT_BYTES },
};
```

因此首先要调用`virt_set_memmap(vms)`初始化`vms`，使其获得各部分地址空间信息。

```c
VirtMachineState *vms = VIRT_MACHINE(machine);
if (!vms->memmap) {
	virt_set_memmap(vms);
}
```

之后通过`vms`结构就能方便初始化各部分地址空间。如当要为虚拟机添加内存时可以通过`memory_region_add_subregion()`将代表虚拟机内存的MemoryRegion`machine->ram`添加到系统内存地址空间的`vms->memmap[VIRT_MEM].base`处

当要创建gic设备时可以通过`create_gic(vms)`函数，传入vms结构完成。

同理要创建pcie设备时也要传入`vms`结构，通过调用`create_pcie(vms)`完成。

其他设备内存空间的创建方法大同小异，都需要vms结构，只是有的可能会需要一些外参数。

如果要自定义地址空间布局则可以通过创建类似的`MemMapEntry`结构实现


# powerpc

以`g3beige`为例，`g3beige`类型的虚拟机的初始化函数为`ppc_heathrow_init`，其内存初始化部分主要如下：

```c
static void ppc_heathrow_init(MachineState *machine)
{
	...
    /* allocate RAM */
    if (ram_size > 2047 * MiB) {
        error_report("Too much memory for this machine: %" PRId64 " MB, "
                     "maximum 2047 MB", ram_size / MiB);
        exit(1);
    }

	memory_region_add_subregion(get_system_memory(), 0, machine->ram);

	/* allocate and load firmware ROM */
    memory_region_init_rom(bios, NULL, "ppc_heathrow.bios", PROM_SIZE,
                           &error_fatal);
    memory_region_add_subregion(get_system_memory(), PROM_BASE, bios);
	...
}
```

首先`g3beige`会限制内存的大小，如果内存大于2047MB则会报错退出

```c
if (ram_size > 2047 * MiB) {
	error_report("Too much memory for this machine: %" PRId64 " MB, "
				 "maximum 2047 MB", ram_size / MiB);
	exit(1);
}
```

之后让ram`machine->ram`从地址空间0开始：

```c
memory_region_add_subregion(get_system_memory(), 0, machine->ram);
```

根据宏`PROM_SIZE`(4MB)创建对应大小的rom作为bios

```c
memory_region_init_rom(bios, NULL, "ppc_heathrow.bios", PROM_SIZE, &error_fatal);
```

然后根据宏`PROM_BASE`(0xffc00000)，设置其起始地址

```c
memory_region_add_subregion(get_system_memory(), PROM_BASE, bios);
```

## mac99

再看mac99类型虚拟机的初始化方法，其初始化主函数为`ppc_core99_init`

```c
static void ppc_core99_init(MachineState *machine)
{
	...
	/* allocate RAM */
    memory_region_add_subregion(get_system_memory(), 0, machine->ram);

    /* allocate and load firmware ROM */
    memory_region_init_rom(bios, NULL, "ppc_core99.bios", PROM_SIZE,
                           &error_fatal);
    memory_region_add_subregion(get_system_memory(), PROM_BASE, bios);
	...
}
```

同`g3beige`类型的虚拟机，其也是直接将ram分配到地址0，将`PROM_SIZE`大小的bios rom分配到地址`PROM_BASE`。


# 总结

不同类型虚拟机的地址空间布局有所不同，内存空间的划分笔者认为主要有两部分，划分ram和划分rom。但不论是ram还是rom其划分方法是一样的：

- 确定管理这块空间的MemoryRegion结构的大小，之后通过`memory_region_init_`系列函数初始化，初始化的mr管理ram则用`memory_region_init_ram`，初始化的mr管理rom则用`memory_region_init_rom`初始化
- 确定改MemoryRegion结构在地址空间的起始位置，之后通过`memory_region_add_subregion`加到对应的空间

如mac99类型中rom的大小为`PROM_SIZE`，就通过下面方式初始化其对应的MemoryRegion

```c
memory_region_init_rom(bios, NULL, "ppc_core99.bios", PROM_SIZE, &error_fatal);
```

第一个参数的待初始化的MemoryRegion，第二个参数为该mr的owner，第三个参数为该mr的名称，第四个参数为该mr的大小，最后一个参数用于记录报错信息

rom在地址空间的位置为`PROM_BASE`，则通过下面方法添加到对应地址空间

```c
memory_region_add_subregion(get_system_memory(), PROM_BASE, bios);
```

第一个参数为被加入到的MemoryRegion，第二个参数为待加入的mr在被加入mr中的位置，第三个参数为待加入的mr




