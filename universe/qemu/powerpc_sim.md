---
title: PowerPC虚拟机模拟的开发方法
date: 2021-07-24
tags: 
- PowerPC
- qemu
mathjax: true
---

# 模拟方法概述

先创建一个最小化的开发板，其应该具有基本的io、屏幕回显能力。之后根据镜像加载、启动过程中的报错信息，对这个最小化开发板进行修补：添加设备、修改设备树等。直至内核成功启动。

创建虚拟机可以分为如下5个阶段，伪码描述如下：

```c
static void my_machine_init(MachineState *machine){
	create_cpu();
	create_mem();
	create_dev();
	load_kernel();
	create_device_tree();
}
```

首先`create_cpu`和`create_mem`阶段为虚拟机创建基本的处理和储存单元，但此时缺少serial设备还不具备屏幕回显的能力，因此需要在`create_dev`阶段创建serial设备。`create_dev`阶段还会创建虚拟机需要的其他设备，但这里先从最小化开始，发现确实设备再进行设备补充。

`load_kernel()`阶段负责处理内核加载过程，由于qemu不会同一的处理内核加载的方法，这里我们需要在源码中显式地编写内核、固件等的加载逻辑，方便的是qemu提供了ELF文件加载的接口函数`load_elf()`系列函数，我们可以专注于加载逻辑的处理。之后让虚拟机CPU的pc寄存器指向程序入口。这样在虚拟机启动后cpu将会从pc寄存器指向的位置开始顺序执行。

最后在`create_device_tree`阶段为cpu准备设备树。设备树就相当于开发板上硬件的使用说明，告诉cpu设备都在什么位置，e500核心的cpu就使用`gpr[3]`寄存器存储设备树基地址。


# QEMU中的创建虚拟开发板的流程

## 基础功能准备

这个部分对应虚拟机创建的`create_cpu`、`create_mem`和`create_dev`阶段，主要功能是：创建cpu和memory提供图灵完备;创建serial设备，让启动信息能打印到屏幕，保证人机交互。


## 内核加载启动流程

在`load_kernel`阶段，通过`load_elf()`接口可以将elf文件加载到虚拟机内存中。同时通过其返回的`lowaddr`和`highaddr`可以计算出文件的入口地址，将pc寄存器指向之则程序准备就绪。

```c
int load_elf(const char *filename,
             uint64_t (*elf_note_fn)(void *, void *, bool),
             uint64_t (*translate_fn)(void *, uint64_t),
             void *translate_opaque, uint64_t *pentry, uint64_t *lowaddr,
             uint64_t *highaddr, uint32_t *pflags, int big_endian,
             int elf_machine, int clear_lsb, int data_swab)
```


## 加载/创建设备树

qemu可以通过`-dtb <dtb_file>`命令加载设备树，前提是源码中有对应的处理。即如果我们需要通过`-dtb <file>`的方式加载dtb文件，需要在源码中显式添加处理逻辑，因为`-dtb`参数的解析不属于qemu命令行解析的通用部分，具体实现由模拟板子的代码决定。

`-dtb`参数可以通过`qemu_opt_get(machine_opts, "dtb");`获取，之后通过`load_device_tree()`遍完成了将`.dtb`文件解析成设备树对象的过程。

```c
QemuOpts *machine_opts = qemu_get_machine_opts();
const char *dtb_file = qemu_opt_get(machine_opts, "dtb");

if (dtb_file) {
	char *filename;
	filename = qemu_find_file(QEMU_FILE_TYPE_BIOS, dtb_file);
	fdt = load_device_tree(filename, &fdt_size);
}
```

而设备树文件从何而来呢？一种方法是通过编写dts文件，然后编译生成dtb文件。这里主要利用另一种方法：先利用qemu创建设备树的接口创建设备树，再利用qemu导出dtb文件的功能(详见后面dumpdtb处理部分内容)将设备树dtb文件从内存中导出。

利用qemu提供的接口创建设备树的方法如下：首先通过`void* fdt = create_device_tree(&fdt_size);`创建空设备树对象。之后设备树的内容就可以根据具体情况自定义了。qemu提供如下常用API：

- `qemu_fdt_add_subnode(fdt, "node_name")`，创建新节点
	* 空设备树对象只存在根节点`/`，每个新设备的加入都得先创建新节点
- `qemu_fdt_setprop_string(fdt, "/path/to/node", "key", "value")`，为节点添加字符串类型的键值对
- `qemu_fdt_setprop_cell(fdt, "/path/to/node", "key", number)`，为节点添加数字类型的键值对

最后还有将创建好的设备树对象写入虚拟机的内存，供虚拟机使用，这个步骤可以通过`cpu_physical_memory_write(addr, fdt, fdt_size);`完成，其中addr就是加载elf阶段计算出的设备树地址。


## 内核启动与设备支持

假设你新购入了一台高科技设备，你完全不知道怎么使用，你就得查看这个设备提供的使用手册。你就相当于CPU，使用手册就相当于设备树。当你根据手册想要操作某个功能时，你发现这个机器上并不存在这样的设备，这时你一定会感到诧异。

同理内核启动时根据设备树取得不到对应的设备支持，则启动无法进行。所以在QEMU中还需要将设备模拟出来。

qemu提供了许多设备的QOM，我们可以通过QOM方便的将设备创建，而具体设备之间怎么连接、地址空间怎么划分就得根据具体手册决定了。


# 模拟实例及容易掉入的坑

我模拟开发板的思路如下：

- 1. 创建一个最小化的机器，使之能够进入到加载内核阶段
- 2. 根据加载内核时的报错信息添加必要设备


## 最小化尝试

仅创建CPU和内存，根据图灵机的原理，一个最小化的机器应该能处理和保存状态，对应的需要CPU和Memory。


### 创建CPU

CPU也可以通过`object_new()`由QOM创建，然后通过`qdev_realize_and_unref()`执行CPU的实例化，如：

```c
cpu = YOUR_CPU(object_new("cpu_type"));
qdev_realize_and_unref(DEVICE(cpu), NULL, &error_fatal);
```

这里面省略了很多细节，如:设置cpu状态、中断引脚、时钟频率、mmu类型等。这里需要注意的是，必须为cpu注册reset事件，必须为cpu做时钟频率初始化。


#### 注意事项1：为cpu注册reset事件

分析qemu初始化的主流程，即`qemu_init()`部分的代码，并通过gdb调试函数触发状况发现：

```
qemu_init(){
	…
	machine_run_board_init()
	…
	qemu_system_reset()
	…
}
```

qemu初始化会先在`machine_run_board_init()`中执行虚拟机基本的创建，之后会在`qemu_system_reset()`调用通过`qemu_register_reset()`注册的reset事件。

cpu的reset事件会设置cpu初始状态下各个寄存器的值，如会设置表示程序入口的pc寄存器。如果不进行明确设置，而程序的入口会是一个无法确定的位置，这时我们不希望的。

为CPU注册reset事件的代码如下，通过`qemu_register_reset()`注册，在reset阶段qemu就会调用`ppce500_cpu_reset(cpu)`对cpu进行reset。

```c
qemu_register_reset(ppce500_cpu_reset, cpu);
```

这个步骤的目的是设置cpu的寄存器、cpu状态等。上面在加载elf阶段所说的将pc寄存器指向elf文件入口，将gpr[3]寄存器指向设备树地址就是在这里完成。`ppce500_cpu_reset()`代码如下：

```c
static void ppce500_cpu_reset(void *opaque)
{
    PowerPCCPU *cpu = opaque;
    CPUState *cs = CPU(cpu);
    CPUPPCState *env = &cpu->env;
    struct boot_info *bi = env->load_info;

    cpu_reset(cs);

    /* Set initial guest state. */
    cs->halted = 0;
    env->gpr[1] = (16 * MiB) - 8;
    env->gpr[3] = bi->dt_base;
    env->gpr[4] = 0;
    env->gpr[5] = 0;
    env->gpr[6] = EPAPR_MAGIC;
    env->gpr[7] = mmubooke_initial_mapsize(env);
    env->gpr[8] = 0;
    env->gpr[9] = 0;
    env->nip = bi->entry;
    mmubooke_create_initial_mapping(env);
}
```


#### 注意事项2：为cpu做时钟频率初始化

在PowerPC开发板模拟中可以通过调用`ppc_booke_timers_init(cpu, freq, flags);`完成


### 内存创建

QEMU中使用MemoryRegion结构管理内存，内存创建可以分为3步：

- 1 创建MemoryRegion结构mr用于管理内存
- 2 对mr进行初始化，如初始化大小、类型等
- 3 为mr指定地址空间，这样一来就使mr对这部分空间进行管理

首先，MemoryRegion结构可以通过`MemoryRegion *mr = g_new0(MemoryRegion, 1);`创建。

之后，对于内存初始化，qemu提供了很多api如`memory_region_init_ram()`、`memory_region_init_rom()`、`memory_region_init_alias()`等。

最后通过`memory_region_add_subregion(system_memory, base, mr)`，指定mr管理`system_memory`的base处开始的空间。

这里最小化阶段仅对ram进行创建，由因为qemu创建虚拟机的公共部分会完成ram的创建和初始化，所有这里仅用对ram进行空间分配：

```c
memory_region_add_subregion(system_memory, 0, machine->ram);
```


## 最小化改进

最小化的虚拟开发板创建完成后发现无法启动，经过与其他qemu现成开发板的比较，发现有如下原因：

- 1 没有serial设备，无法将文字显示到屏幕
	* 与`openrisc_sim`开发板比较，发起其创建的结构是：cpu, memory, serial, load kernel。所以猜测缺少serial设备屏幕无法回显
- 2 没有加载内核，cpu相当于什么都没干
- 3 没有创建设备树，内核"迷路"了


### serial设备创建

使用`serial_mm_init()`函数可以完成serial-mm设备的创建，其原型如下：

```c
SerialMM *serial_mm_init(MemoryRegion *address_space,
                         hwaddr base, int regshift,
                         qemu_irq irq, int baudbase,
                         Chardev *chr, enum device_endian end)
```

- 参数说明
	* `address_space`，表示该设备的所在的MemoryRegion
	* `base`，表示访问该设备的地址(相当第一个参数来说)
	* `regshift`
	* `irq`，中断相关
	* `baudbase`，不确定，根据名称的猜测：设备需要协商波特率才能相互通信吧
	* `chr`，不确定，唯一表示设备的结构，可以通过`serial_hd()`函数生成
	* `end`，指明的设备是大端模式`DEVICE_BIG_ENDIAN`还是小端模式`DEVICE_LITTLE_ENDIAN`或者`DEVICE_NATIVE_ENDIAN`模式

如下面的步骤就完成了创建serial-mm设备并将其地址为设置系统内存空间的0x84004500处的操作：

```c
serial_mm_init(get_system_memory(), 0x84004500, 0, NULL,
                   399193, serial_hd(0), DEVICE_BIG_ENDIAN);
```


#### 注意事项1：通过设备树告之stdout输出到serial设备

设备树的`/chosen`节点可以控制内核启动的一些行为，如`bootargs`属性的值会作为内核的启动参数。这里需要用到的是`stdout-path`属性，其指定标准输出到什么设备。我们要让屏幕回显，则让标准输出到serial设备。如：

```c
qemu_fdt_setprop_string(fdt, "/chosen", "stdout-path", "/serial@0xXXXXX");
```


### 加载内核/bios

首先，可以通过`machine->kernel_filename`和`machine->firmware`分别获取到`-kernel`和`-bios`参数的内容。

```c
kernel_name =  machine->kernel_filename;
bios_name =  machine->firmware;
```

之后通过`char *filename = qemu_find_file()`获取文件路径。

然后可以通过`load_elf()`将elf文件载入虚拟机内存，如：

```c
payload_size = load_elf(filename, NULL, NULL, NULL,
						&bios_entry, &loadaddr, NULL, NULL,
						1, PPC_ELF_MACHINE, 0, 0);
```

通过其返回的地址，这里是`bios_entry`和`load_addr`可以计算出PC寄存器应该指向的位置，将PC寄存器指向之。


#### 通过bios间接加载内核

QEMU中的ppce500虚拟机在没有指定`-kernel`和`-bios`参数时默认会加载u-boot作为bios，再由u-boot加载内存。而在GRBP2020Kit开发手册中使用的也是uboot加载内核的方式，所有有必要了解uboot的基本使用：

从内存中加载内核:

```
bootm <uImage_addr> [initrd_addr] [dtb_addr]
```

如果不需要`initrd`镜像而需要传入设备树文件，可以通过`-`传递空参数。如`bootm 0x2 - 0x4`，第二个参数的位置为`-`表示无`initrd`传入

查看设备树信息：

首先需要使用`fdt addr <addr> [len]`命令告知设备树信息的位置，以让fdt命令识别设备树。

```
fdt addr 0x800000
```

如上述命令告知fdt设备树文件位于`0x800000`处，之后可以使用fdt对设备树进行读写等操作，如：

- `fdt print <path>`，打印以`<path>`为根节点的完整子树信息
- `fdt list <path>`，打印以`<path>`为根节点的子节点
- `fdt set <path> <prop> [val]`，设置设备树属性


### 为内核/bios创建设备树

像之前说的，可以通过`-dtb`加载设备树，也可以手动创建设备树。对过程总结以下就是：

- 创建设备树对象
	* 可能用到`load_device_tree`或`create_device_tree`
- 自定义设备树对象
	* 可能用到`qemu_fdt_add_subnode`等
- 将设备树写人虚拟机内存`cpu_physical_memory_write(addr, fdt, fdt_size);`

下面列出手动创建设备树的一些注意事项。


#### 注意事项1：soc节点

PowerPC虚拟机的设备树需存在soc节点，即使节点为空。


#### 注意事项2：设置设备树的兼容性

设备树兼容性属性的设置非常重要，不同配置的内核不会识别兼容性不合要求的设备树。因为控制compatible为单一变量测试发现，当根节点的compatible属性设置不当时，虚拟机启动失败。

设备树的兼容主要主要通过设置根节点`/`的`model`和`compatible`属性完成，如：

```c
const char model[] = "QEMU ppce500";
const char compatible[] = "fsl,qemu-e500";
qemu_fdt_setprop(fdt, "/", "model", model, sizeof(model));
qemu_fdt_setprop(fdt, "/", "compatible", compatible,
				 sizeof(compatible));
```

如这里设置兼容e500，所以为P2020编译的镜像不会使用。要使其同时兼容e500和P2020可以做如下修改：

```
const char model[] = "fsl,P2020RDB\0QEMU ppce500";
const char compatible[] = "fsl,P2020RDB\0fsl,qemu-e500";
qemu_fdt_setprop(fdt, "/", "model", model, sizeof(model));
qemu_fdt_setprop(fdt, "/", "compatible", compatible,
				 sizeof(compatible));
```


#### 注意事项3：设备树节点的顺序

设备树节点的顺序也是非常重要，因为内核经常将第一个设备作为一些操作的默认设备，如根据设备树的第一cpu作为主cpu、将第一个设备作为serial设备等。

需要主要的是，qemu中`qemu_fdt_add_subnode`创建设备节点的顺序和设备树生成后节点的顺序相反。如：有两个cpu，cpu0和cpu1，要使cpu0作为执行的主cpu，则代码书写是顺序应该是先加cpu1再加cpu0：

```c
/* We need to generate the cpu nodes in reverse order, so Linux can pick
       the first node as boot node and be happy */
qemu_fdt_add_subnode(fdt, "/cpu1");
qemu_fdt_add_subnode(fdt, "/cpu0");
```

同样的还有serial设备官方描述如下：

```c
/*
 * We have to generate ser1 first, because Linux takes the first
 * device it finds in the dt as serial output device. And we generate
 * devices in reverse order to the dt.
 */
```


#### 注意事项4：为创建设备树注册reset事件

同为cpu注册reset事件，要用`qemu_register_reset()`为创建设备树reset事件。


#### 注意事项5：要让模拟的开发板至此dumpdtb需要额外处理

要让虚拟中支持dumpdtb从而得到我需要的设备树dtb文件，我们也需要显式的对其进行处理，这样过程比较简单，可以通过调用`qemu_fdt_dumpdtb(fdt, fdt_size)`完成，这时当`-M <machien>[,dumpdtb=<file.dtb>]`，dumpdtb参数存在时qemu会将设备树信息保存到`file.dtb`


## 最小化补完

内核启动阶段，通过报错信息发现缺少如下设备内核无法启动：

- pic
- pci

所以需要在第一章整体流程中的create_dev()阶段补充相应的设备


