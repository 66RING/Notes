---
title: PowerPC虚拟机模拟的开发方法
date: 2021-07-24
tags: 
- PowerPC
- qemu
mathjax: true
---

# 模拟方法概述

创建虚拟机可以分为如下5个阶段，伪码描述如下：

```c
static void my_machine_init(MachineState *machine){
	create_cpu
	create_mem
	create_dev
	load_kernel
	create_device_tree
}
```

`create_cpu`阶段主要负责cpu创建的内容，包括cpu对象创建、cpu reset方法注册、cpu频率初始化等操作。

`create_mem`阶段主要负责内存创建方面的内容，如ram、rom等创建和初始化。

`create_dev`阶段主要负责虚拟机其他设备和各种控制器的创建工作，包括pci总线、中断控制器、网卡等。

`load_kernel`阶段负责将内核镜像载入虚拟机内存，供虚拟机启动时使用。

`create_device_tree`阶段为内核准备设备树。因为powerpc架构的机器启动是基于设备树的，所以设备树的构造十分关键，这关系到内核能否找到相应设备。

为了让qemu能够识别到我们创建的虚拟机，我们还需要使用qemu提供的接口向qemu注册：

```
static void p2020_machine_class_init(MachineClass *mc)
{
    mc->desc = "GRB-P2020 board";
    mc->init = p2020_machine_init;
    mc->max_cpus = 2;
    mc->default_cpu_type = POWERPC_CPU_TYPE_NAME("e500v2_v30");
    mc->default_ram_id = "mpc8544ds.ram";
}


#define TYPE_P2020_MACHINE  MACHINE_TYPE_NAME("p2020")

static const TypeInfo p2020_info = {
    .name          = TYPE_P2020_MACHINE,
    .parent        = TYPE_PPCE500_MACHINE,
    .class_init    = p2020_machine_class_init,
    .interfaces    = (InterfaceInfo[]) {
         { TYPE_HOTPLUG_HANDLER },
         { }
    }
};

static void p2020_register_types(void)
{
    type_register_static(&p2020_info);
}
type_init(p2020_register_types)
```


# QEMU中的创建虚拟开发板的流程

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

最小化的虚拟开发板创建完成后发现无法启动，经过与其他qemu虚拟机的比较，发现有如下原因：

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

至此第一章涉及的5个阶段基本完成。可以尝试启动虚拟机，然后进行下一步开发。

```c
static void my_machine_init(MachineState *machine){
	create_cpu
	create_mem
	create_dev
	load_kernel
	create_device_tree
}
```

用如下命令启动，然后内核给我们提供了一些错误信息：

```
qemu-system-ppc  \
  -M p2020, \
  -m 512M \
  -kernel ./vmlinux \
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/ppc_emulation/kernel_bug.png" ></img>

似乎是在说：内核检测设备时检测不到中断控制器。所以这就引出了虚拟机创建的最后一步：**最小化补完，补充内核启动需要的其他设备** 。

有了前面添加serial设备的经验，我在`create_dev`阶段向虚拟机添加pci总线和pic中断控制器，再将设备信息写入设备树：

```
 	// 添加设备部分
	...
    mpicdev = ppce500_init_mpic_a(pms, ccsr_addr_space, irqs);
	...
	dev = qdev_new("e500-pcihost");
    object_property_add_child(qdev_get_machine(), "pci-host", OBJECT(dev));
    qdev_prop_set_uint32(dev, "first_slot", 0x1);
    qdev_prop_set_uint32(dev, "first_pin_irq", pci_irq_nrs[0]);
    s = SYS_BUS_DEVICE(dev);
    sysbus_realize_and_unref(s, &error_fatal);
    for (i = 0; i < PCI_NUM_PINS; i++) {
        sysbus_connect_irq(s, i, qdev_get_gpio_in(mpicdev, pci_irq_nrs[i]));
    }

    memory_region_add_subregion(ccsr_addr_space, MPC8544_PCI_REGS_OFFSET,
                                sysbus_mmio_get_region(s, 0));

	// 填写设备树部分
	char *mpic;
    uint32_t mpic_ph;
    mpic = g_strdup_printf("%s/pic@%llx", soc, MPC8544_MPIC_REGS_OFFSET);
    qemu_fdt_add_subnode(fdt, mpic);
    qemu_fdt_setprop_string(fdt, mpic, "device_type", "open-pic");
    qemu_fdt_setprop_string(fdt, mpic, "compatible", "fsl,mpic");
    qemu_fdt_setprop_cells(fdt, mpic, "reg", MPC8544_MPIC_REGS_OFFSET,
                           0x40000);
    qemu_fdt_setprop_cell(fdt, mpic, "#address-cells", 0);
    qemu_fdt_setprop_cell(fdt, mpic, "#interrupt-cells", 2);
    mpic_ph = qemu_fdt_alloc_phandle(fdt);
    qemu_fdt_setprop_cell(fdt, mpic, "phandle", mpic_ph);
    qemu_fdt_setprop_cell(fdt, mpic, "linux,phandle", mpic_ph);
    qemu_fdt_setprop(fdt, mpic, "interrupt-controller", NULL, 0);
```

再次编译启动，成功进入内核启动阶段，虚拟机开发完成

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/ppc_emulation/booting.png" ></img>

虚拟机源码可在附录p2020文件的src文件夹中获取。放入`./qemu/hw/ppc`文件夹内，编译安装qemu即可。


# 内核启动

内核启动需要配置恰当的内核镜像和根文件系统。这里使用vmlinux + initrd的方式启动内核。

## 环境安装

内核镜像编译需要一些交叉编译的工具，我们这里使用eldk工具链来编译内核。

首先去DENX官网下载对应工具：[The Embedded Linux Development Kit (ELDK)](https://ftp.denx.de/pub/eldk/)。这里eldk使用的版本是5.6，需要powerpc架构的工具链，所以下载[eldk-5.6-powerpc.iso](https://ftp.denx.de/pub/eldk/5.6/iso/eldk-5.6-powerpc.iso)的镜像。

挂载`eldk-5.6-powerpc.iso`，这里挂载到`/mnt`目录

```sh
$ mount /path/to/eldk-5.6-powerpc.iso /mnt
$ cd /mnt
```

执行安装

```sh
./install.sh -d /path/to/install_dir -s toolchain-xenomai-qte -r qte-xenomai-sdk powerpc
```

- `-d`指定工具链的安装目录。如果不指定，默认安装到`/opt/eldk`
- `-s`表示SDK使用的镜像源
- `-r`表示目标使用的RFS镜像源
- 最后一个参数表示安装的目标指令集架构，这里是`powerpc`

更多参数详情查看`./install -h`

工具链就安装到了`/path/to/install_dir`目录了，使用时将需要的二进制文件添加到环境变量即可，如：

```
export PATH=$PATH:/path/to/install_dir/powerpc/sysroots/i686-eldk-linux/usr/bin:/path/to/install_dir/powerpc/sysroots/i686-eldk-linux/usr/bin/powerpc-linux:/path/to/install_dir/powerpc/sysroots/powerpc-linux/usr/bin/powerpc-linux
```


## 内核编译

这里内核的版本是linux-denx-3.16D20180629，可以在p2020附件中获取：


```
tar xvf linux-denx-3.16D20180629.tar.bz2
cd linux-denx-3.16
make clean
unset LDFLAGS
export ARCH=powerpc
exprot CROSS_COMPILE=powerpc-linux-
make menuconfig //进入 linux 内核配置
```

ARCH表示编译的目标架构，CROSS\_COMPILE表示交叉编译工具的前缀

a,选择以下配置

```
General setup --->
	[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
Device Drivers--->
	[*] Memery Technology Device(MTD) support--->
		RAM/ROM/Flash chip drivces--->
			[*] support for intel/sharp chips
		Mapping drivers for chip access ---> 
			[*] CFI Flash device mapped on P2020GRB

```

b,同时将 File systems---> Miscellaneous filesystems--->中的 JFFS2 文件系统选中，保存退出

执行编译

```
make
```

在源码根目录可以获得vmlinux镜像文件、在目录 linux-denx-3.16/arch/powerpc/boot中，可以获得其他格式的镜像文件。


### ld: scripts/dtc/dtc-parser错误

编译时可能会遇到`ld: scripts/dtc/dtc-parser`错误，这是设备树生成脚本导致的。修改`./linux-denx-3.16/scripts/dtc/dtc-lexer.lex.c`文件640行处，为`extern YYLTYPE yylloc;`，再次make即可

```
- YYLTYPE yylloc;
+ extern YYLTYPE yylloc;
```


## 根文件系统编译

这里根文件系统使用busybox。编译busybox最好使用较高版本的工具链，不然可能遇到未知的错误，这时使用musl提供的工具链。

musl.cc提供的工具链安装步骤：

- 下载需要的[工具链](https://musl.cc/#binaries)
- 解压缩
- 添加环境变量

busybox使用1.33版本，可在附录p2020文件中获得。

```
tar -xjvf busybox-1.33.0.tar.bz2
cd busybox-1.33
export ARCH=powerpc
exprot CROSS_COMPILE=powerpc-linux-musl-
```

选择静态链接：

```
make menuconfig

Busybox Settings --->
	Build Options --->
		[*] Build BusyBox as a static binary (no shared libs)
```

编译安装：

```
make
make install
```

编译生成的文件在`_install`目录，检查以下是否是目标架构：

```
$ cd _install
$ ls
bin sbin user linuxrc

$ file ./bin/busybox
./bin/busybox: ELF 32-bit MSB executable, PowerPC or cisco 4500, version 1 (SYSV), statically linked, stripped

```

如果不是PowerPC，不是静态链接`statically linked`就需要检查过程哪里处理差错需要重新编译。


## 制作initrd

要让内核从initrd启动需要将根文件系统打包成cpio文档。

这里介绍一种最小化的根文件系统，这个根文件系统仅负责进入命令行。完整的根文件系统创建方法可以查阅互联网。

```
$ cd _install
$ ls
bin sbin user linuxrc
```

创建一个`init`脚本作为内核启动的init程序，内容如下：

```
#!/bin/sh

echo "###############################"
echo "######### Hello World #########"
echo "###############################"

exec /bin/sh
```

之后使用如下命令创建cpio文档：

```
find . -print0 | cpio --null -ov --format=newc | gzip -9 > initrd.gz
```

这样就得到了initrd文件：initrd.gz


## 虚拟机启动

我们从0创建了内核镜像`vmlinux`和根文件系统initrd：`initrd.gz`，使用如下命令启动：

```
qemu-system-ppc  \
  -M p2020, \
  -m 512M \
  -kernel ./vmlinux \
  -initrd ./initrd.gz \
  -append "rdinit=/init"
```

`-append`表示内核的启动参数，告诉内核使用initrd启动，init程序位置为`/init`

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/ppc_emulation/bootup.png" ></img>

镜像文件、根文件系统initrd和启动脚本start.sh可在p2020附录的image文件夹中获取。


# 使用flash设备作为外存

我们发现在qemu中pflash设备的创建和参数化方法比较简单，仅需如下代码即可实现。于是为了简单起见，我们打算用pflash设备来模拟其他外存设备。

```c
// 创建
DeviceState *dev = qdev_new(TYPE_PFLASH_CFI01);
qdev_prop_set_uint64(dev, "sector-length", VIRT_FLASH_SECTOR_SIZE);
qdev_prop_set_uint32(dev, "num-blocks", memmap[P2020_FLASH].size / VIRT_FLASH_SECTOR_SIZE);
qdev_prop_set_uint8(dev, "width", 2);
qdev_prop_set_uint8(dev, "device-width", 2);
qdev_prop_set_bit(dev, "big-endian", true);
qdev_prop_set_string(dev, "name", "flash");
PFlashCFI01 *flash = PFLASH_CFI01(dev);

// 参数化
pflash_cfi01_legacy_drive(flash, drive_get(IF_PFLASH, 0, 0));

// 分配空间
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
memory_region_add_subregion(system_memory, memmap[P2020_FLASH].base,
                            sysbus_mmio_get_region(SYS_BUS_DEVICE(dev), 0));
```

不过qemu会检查磁盘文件大小和flash设备的大小，当flash设备大小与磁盘文件大小一直时才能启动虚拟机。

这里提供了一个jffs2格式的磁盘文件：`rootfs.jffs2`，可以在附录p2020的image文件夹中获取。如果需要自行自作jffs2格式的文件系统，可以使用附录p2020的tools文件夹中的`mkfs.jffs2`工作制作。这里仅演示从jffs2磁盘文件启动的方法。

前面说得磁盘文件要与flash设备大小一致，我们可以使用`fallocate`工具为文件扩容。这里我们的flash设备大小为16M，所以：

```
$ cp rootfs.jffs2 rootfs.jffs2.flash
$ fallocate -l 16M rootfs.jffs2.flash
```

之后使用如下命令启动：

```
qemu-system-ppc  \
  -M p2020, \
  -m 512M \
  -kernel ./vmlinux \
  -drive file=./rootfs.jffs2.flash,if=pflash  \
  -append "root=/dev/mtdblock0 rootfstype=jffs2 rw rdinit=/linuxrc" \
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/ppc_emulation/grb.png" ></img>

原始`rootfs.jffs2`、重新分配大小的`rootfs.jffs2.flash`以及从flash启动的启动脚本`start_with_jffs2.sh`见附录p2020的image文件夹。


## 让flash更加通用

既然我们要用flash设备作为外存使用，那它应该还有支持其他文件系统，如ext2、ext3、ext4等。

但当我们打算从ext3格式的磁盘文件启动时，内核报错:

```
qemu-system-ppc  \
  -M p2020, \
  -m 512M \
  -kernel ./vmlinux \
  -drive file=./rootfs.ext3,if=pflash  \
  -append "root=/dev/mtdblock0 rw rdinit=/linuxrc" \


EXT3-fs error (device mtdblock0): ext3_get_inode_loc: unable to read inode block - inode=2049, block=8260
Starting init: /sbin/init exists but couldn't execute it (error -12)
```

这时由于我们创建的flash设备的block为8192，而ext3文件系统的block为16384导致的，我们可以使用`resize2fs`工具修改ext3的block大小：

```
cp rootfs.ext3 rootfs.ext3.flash
resize2fs rootfs.ext3.flash 8192
```

再使用`fallocate`工具扩大磁盘文件：

```
fallocate -l 16M rootfs.ext3.flash
```

启动成功

```
qemu-system-ppc  \
  -M p2020, \
  -m 512M \
  -kernel ./vmlinux \
  -drive file=./rootfs.ext3.flash,if=pflash  \
  -append "root=/dev/mtdblock0 rw rdinit=/linuxrc" \
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/ppc_emulation/ext_suc.png" ></img>

原始磁盘文件`rootfs.ext3`、修改后的磁盘文件`rootfs.ext3.flash`以及启动脚本`start_with_ext3.sh`见附录p2020的image文件夹。

ext2、ext4文件系统格式的处理方式同理。




