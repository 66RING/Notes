---
title: 使用qemu启动基于open risc的虚拟机
date: 2020-12-05
tags: 
- qemu
mathjax: true
---

# 使用qemu启动虚拟机

## 安装qemu

可能需要的依赖

```
autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev git
```

下载qemu源文件

```
git clone https://github.com/qemu/qemu
```

进入qemu目录`cd qemu`，配置qemu

```
./configure --target-list='or1k-softmmu or1k-linux-user'
```

- `xxx-softmmu`，内存管理单元模拟
- `xxx-user`，用户空间模拟
- 还可以有其他参数如`--enable-debug`、`--enable-debug-info`等

编译安装qemu

```
make -j
make install
```

## 安装基于open risc的linux

### 从源码编译linux

- 1.安装编译所需工具链
    * binaries：https://github.com/openrisc/or1k-gcc/releases
    * building：https://github.com/stffrdhrn/or1k-toolchain-build
    * 其他toolchains：https://openrisc.io/software
    * 或编译好的：https://toolchains.bootlin.com/releases_openrisc.html
- 2.编译指定架构的内核
    * 使用默认配置`make ARCH=openrisc CROSS_COMPILE="or1k-linux-" defconfig`
    * 编译`make ARCH=openrisc CROSS_COMPILE="or1k-linux-"`


### 使用现成的编译好的linux

如果编译安装基于open risc的linux过于费时，可以使用现成的已经编译号的linux

```
wget http://shorne.noip.me/downloads/or1k-linux-5.0.gz
```

解压缩

```
gunzip or1k-linux-5.0.gz
```

## 制作根文件系统

有多种方式制作根文件系统，这里使用busybox制作。

下载busybox源码

```
https://busybox.net/downloads/
```

进入源码目录，配置busybox

```
make CROSS_COMPILE="or1k-linux-" menuconfig
# or1k-linux- 是你目标架构系统需要的工具链
```

进入配置菜单，将`Settings`中的`Build Options`中的`Build static binary (no shared libs)`选中并保存配置，退出。

编译安装

```
make CROSS_COMPILE="or1k-linux-"
make CROSS_COMPILE="or1k-linux-" install
```

之后在源码目录下会多出`_install`文件夹。


### 制作一个最小的文件系统

使用`qemu-img`创建一个分区，并安装文件系统

```
qemu-img create rootfs.img  1g
mkfs.ext4 rootfs.img   // 制作ext4格式的文件系统
```

将`_install`文件夹下的东西拷贝到文件系统中

```
mkdir rootfs
sudo mount -o loop rootfs.img  rootfs
cd rootfs
sudo cp -r ../busyboxsource/_install/* .
sudo mkdir proc sys dev etc etc/init.d
```

另外还要创建一个简单的init的RC文件：

```
cd etc/init.d/
sudo touch rcS
sudo vi rcS
```

编辑该文件内容如下：

```
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

然后修改 rcS 文件权限，加上可执行权限，这样当 busybox 的init 运行起来后，就能运行这个 /etc/init.d/rcS 脚本。

```
sudo chmod +x rcS
```

最后退出 rootfs 目录并卸载文件系统：

```
sudo umount rootfs
```

至此，文件系统就制作完成了。


## 启动虚拟机

使用编译好的qemu启动linux

```
qemu-system-or1k -cpu or1200 -M or1k-sim -kernel or1k-linux-5.0 -serial stdio -nographic -monitor none
```

- 参数说明
    * `qemu-system-`，用于模拟不同的硬件平台，这里是OpenRISC，所以是`qemu-system-or1k`
    * `-cpu`，指定虚拟cpu型号
    * `-M `，指定虚拟机类型
    * `-m `，指定虚拟机内存大小
    * `-kernel`，指定内核镜像
    * `-serial`，重定向串口到宿主机的设备
    * `-nographic`，通常，QEMU使用SDL显示VGA输出，使用这个选项，使qemu成为简单的命令行应用程序
    * `-monitor`，重定向显示器到宿主机的设备
    * `-S`，在内核入口打断点
    * `-s`，等待gdb连接，默认开启1234端口


# qemu工作原理

## qemu虚拟机原理

利用CPU硬件提供的虚拟化支持，做硬件加速，如果虚拟机(guest)和宿主机(host)的架构相同，虚拟机的指令可以直接在host运行，在将结果告诉guest。这里是用上了kvm:linux内核的虚拟化支持，如果遇到kvm无法处理的操作时(如io处理)会退出kvm并交给qemu处理。

qemu处理，那要将guest机的指令翻译成host机器的指令然后在host机执行，最后再将执行结果告诉guest机。


## qemu模拟硬件原理

计算机本质上就是一个能在特定位置读写并作出响应的机器。只要通过软件模拟这个读写过程，并且作出响应的相应。那么在虚拟机看来，它就拥有一个实实在在的硬件。这就涉及到qemu的QOM。


## QOM, qemu object module

QOM创建可以分为四步

- 将TypeInfo注册TypeImpl
- 实例化ObjectClass
    * The base for all classes.
- 实例化Object
    * The base for all objects.
- 添加Property

总的来说，根据TypeInfo创建TypeImpl ，然后根据TypeImpl创建对应的ObjectClass，再根据TypeImpl创建对应的Object，ObjectClass和Object都有自己的Property


### 自定义Object

```c
typedef struct MyDevice
{
    DeviceState parent;  //父对象必须是该对象数据结构的第一个属性，以便实现父对象向子对象的cast

    int reg0, reg1, reg2;  // 定义设备的寄存器
} MyDevice;
```


#### QOM中的多态

**C中的继承的一种实现方式：父类对象为结构体的第一成员** ，C标准确保结构体的第一个成员位于0字节，可以直接进行类型转换。

看下面一段代码

```c
typedef struct ObjectClass
{
    Type type;
    // ...
}ObjectClass;

typedef struct Object
{
    ObjectClass *class;  // 没有祖父，所以可以用指针
    // ...
}Object;

typedef struct MyDevice {
    Object parent_obj;
    int reg0, reg1;
} PCITestDevState;
```

从父类到子类的关系是`ObjectClass->Object->MyDevice`，在内存中是这样的：

![inheritance](./inheritance.jpg)

因此在要类型转换时取相应大小的内存就能得到对应的类。如从子类转变为父类或祖父类。


### ObjectClass注册与初始化

TypeImpl的创建，用户通过定义TypeInfo类型，然后调用`type_register`或`type_register_static`调用`type_init`宏注册到全局hash表中，最后通过定义的`TyepInfo`类型创建TypeImpl对象。如：

```c
static const TypeInfo my_device_info = {
    .name = "name",  // 类名，作为hash表的key，value就是生成的TypeImpl
    .parent = "parent",  // 父类名
    .instance_size = sizeof(MyDevice),
    .class_size = sizeof(MyDevice),
    .instance_init = my_init,   // 实例(设备)初始化
    .class_init = my_device_class_init, // 类初始化动作，如重写父类函数等
};
static void my_device_register_types(void)
{
    type_register_static(&my_device_info); // 注册，用于生成TypeImpl实例
    // 或者 type_register(&my_device_info);
}
type_init(my_device_register_types)
```

在QEMU启动阶段`vl.c:main()`，注册的函数会在`module_call_init(MODULE_INIT_QOM)`中执行。`name`字段定义我们启动时参数`-device`后面的值


### 类初始化

```c
static type_initialize(TypeImpl *ti){
    // 分配空间等
    // ...
    // 初始化
    ti->class_init(ti->class, ti->class_data);
}
```

通过将自定义`TypeInfo`注册到全局哈希表后，就可以根据哈希表初始化，通过调用`type_initialize`每一个类分配内存空间和初始化(如给`ObjectClass`分配空间和初始化)，如果有`parent`则会递归调用`type_initialize`调用父类的初始化函数

类初始化函数指针由`.class_init`成员接收。`type_initialize`会调用类的`.class_init`成员。相当于 **构造函数** ，但是多数oop语言支持的重写需要我们在该构造函数中修改指针的值。如我们实现了父类的虚拟方法，就可以获取父类引用并给父类成员赋值，一般就是重写父类的虚拟方法，如

```c
void my_device_class_init(ObjectClass *klass, void *class_data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);  // 使用引用修改，原地修改故相当于实现父类虚拟函数，覆盖原有类的方法
    dc->reset = my_device_reset;  // 设备复位
    dc->realize = my_device_realize; // Object初始化函数
}
```

派生类中可以将类原有的虚拟函数指针保存起来，以便以后恢复，保证父类原有的虚拟函数指针不会丢失。

因为C语言保证结构体的第一个成员始于0字节，而所有对象的父类都是其第一个成员，所以我们可以直接进行类型转换。

将一个父类的指针直接转换为子类的指针是不安全的，为了安全校验从Type实例的父类型转换成子类，各类需要提供强制类型转换的宏，如：

```c
DeviceClass *dc = DEVICE_CLASS(klass);
```

这些宏都由`OBJECT_CLASS_CHECK()`封装，其中一个例子如下：

```c
/* include/hw/qdev-core.h */
#define DEVICE_CLASS(klass) OBJECT_CLASS_CHECK(DeviceClass, (klass), TYPE_DEVICE)

/* include/qom/object.h */
/**
 * OBJECT_CLASS_CHECK:
 * @class_type: The C type to use for the return value.
 * @class: A derivative class of @class_type to cast.
 * @name: the QOM typename of @class_type.
 *
 * A type safe version of @object_class_dynamic_cast_assert.  This macro is
 * typically wrapped by each type to perform type safe casts of a class to a
 * specific class type.
 */
#define OBJECT_CLASS_CHECK(class_type, class, name) \
    ((class_type *)object_class_dynamic_cast_assert(OBJECT_CLASS(class), (name), \
                                               __FILE__, __LINE__, __func__))
```

可见安全的类型转换的宏定义通过封装`object_class_dynamic_cast_assert`实现


### Object创建

通过`type_init`，自定义的类型的`ObjectClass`就准备好了，其包含Object初始化需要的一些操作，所以叫`ObjectClass`。

而Type的`Object`实例只有在 **QEMU命令行中添加`-device`选项时才会创建** 。主要是在构造函数重写父类的`(ObjectClass)->realize`，再由`realize`完成主要的Object创建，但有的也会通过`.instance_init`函数指针完成。

`ObjectClass`和`Object`通过`Object`的`class`字段联系。


### 属性

`ObjectClass`和`Object`结构都有一个`GHashTable`类型的`properties`成员，用于存储`属性名`到`ObjectProperty`的映射

```c
typedef struct ObjectProperty
{
    gchar *name;
    gchar *type;
    gchar *description;
    ObjectPropertyAccessor *get;
    ObjectPropertyAccessor *set;
    ObjectPropertyResolve *resolve;
    ObjectPropertyRelease *release;
    void *opaque;
} ObjectProperty;
```


#### 静态属性

静态属性在初始化过程就加入到了`props`中，类对象(`ObjectClass`)中通过调用`object_property_add`函数将属性添加在类实例对象(`Object`)的`properties`成员中。

而类对象`ObjectClass`的`props`成员会在`class_init`中设置。如:

```c
static Property stm32f2xx_usart_properties[] = {
    DEFINE_PROP_CHR("chardev", STM32F2XXUsartState, chr),
    DEFINE_PROP_END_OF_LIST(),
};

static void stm32f2xx_usart_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->reset = stm32f2xx_usart_reset;
    dc->props = stm32f2xx_usart_properties;
    dc->realize = stm32f2xx_usart_realize;
}
```

查看属性:

```sh
qemu-system-<arch> -device <module name>,?
```


#### 动态属性

动态属性是在运行时添加的属性，如用户通过参数传入一个设备，需要作为属性和其他设备关联起来


##### child

`child`实现了组成(composition)关系，表示一个设备(parent)创建了另一个设备(child)，parent掌握child的声明周期，负责向其发送事件。一个device只能有一个parent，但可以有多个child，构成一棵组合树：

通过`object_property_add_child`添加child：

```
object_property_add           
将 child 作为 obj 的属性，属性名name，类型为 "child<child的类名>"，同时getter为object_get_child_property，没有setter
child->parent = obj
```

例如 `x86_cpu_realizefn => x86_cpu_apic_create => object_property_add_child(OBJECT(cpu), "lapic", OBJECT(cpu->apic_state), &error_abort)` 将创建 APICCommonState ，并设置为 X86CPU 的child。


##### link

link实现了backlink关系，表示一个设备引用了另外一个设备，是一种松散的联系。两个设备之间能有多个link关系，可以进行修改。它完善了组合树，使其构成构成了一幅有向图。

通过`object_property_add_link`添加link：

```
创建 LinkProperty ，填充目标(child)的信息
object_property_add
将 LinkProperty 作为 obj 的属性，属性名name，类型为 "link<child的类名>"，同时getter为 object_get_link_property 。如果传入了check函数，则需要回调，设置setter为 object_set_link_property
```

# 参考

- [安装参考](https://wiki.qemu.org/Documentation/Platforms/OpenRISC)
- [使用入门](https://www.jianshu.com/p/e1a4b5b808e0)**






