---
title: qemu中设备实例化过程
author: 66RING
date: 2022-02-24
tags: 
- qemu
mathjax: true
---

# 设备实例化过程：vcpu线程和cpu实例如何对接

## Abstract

qemu设备实例化大致可以分为两个阶段：

1. QOM系统的初始化
2. 具体设备的初始化

纯c编写的qemu为了实现面向对象开发了自己的一套QOM系统(qemu object module)。用户掌握QOM提供的接口后就可以在qemu开发过程中用上一些面向对象的特性。

这里[1.2章](#QOM设备实例化的原理)将先会讨论qemu中用面向对象方式进行实例化的原理，然后在[1.3章](#CPU设备实例化的例子)会通过cpu设备实例化和vcpu线程的启动来进一步说明。


## QOM设备实例化的原理

在OOP语言中，通过构造函数可以得到一个对象实例，这个过程是编译器通过符号表找到了构造方法并调用完成的。那么在纯c编写的qemu中也需要一种方法找到一个对象的构造函数(或者说实例化函数)。在QOM是通过哈希表的方式来维护"符号"到具体功能的映射的。

一个设备需要通过`type_init()`方法注册到QOM系统中，本质上是在哈希表上添加了"对象名"-"注册函数"的键值对。这样就可以通过`object_new("OBJECT_NAME")`来得到对象实例了。

而`type_init()`实际上是利用了编译器参数`__attribute__((constructor))`，这个参数包裹的内容将会在main函数执行前执行。利用这一点，在虚拟机启动前QOM系统就会构造好类对象所需的一些内容，如：构造方法、元类构造方法等。而一个对象的实例化函数则保存在哈希表的`realize`字段中。

这样在我们使用`object_new()`创建一个实例对象时就会可以通过类名来调用之前注册好的对象实例化方法。可以看到`object_new()`接收一个字符串类型的参数用户查表，整个过程会得出如下调用栈，也像许多OOP语言一样QOM会以递归的方式地从父类构造到子类。


```
object_new() -> object_new_with_type() -> object_initialize_with_type()

object_initialize_with_type() {
	type_initialize -> if(!ti->class_init)ti->class_init()  		// 如果类描述信息(metaclass)没初始化则先初始化元类
	object_init_with_type() -> instance_init() -> init instance 	// 利用QOM维护的哈希表调用实例化函数
}
```

在`instance_init()`中会调用`qdev_realize_and_unref()`找到`realized`字段，从而找到一个对象的实例化方法, 调用以完成实例化。

更多关于QOM初始化框架的细节可以参考这[篇文章](https://github.com/66RING/Notes/blob/master/universe/qemu/qemu_initial_framework.md)


## CPU设备实例化的例子

在CPU实例话的过程中会启动一个vcpu线程，这里以ppc的cpu实例化过程为例进行说明。描述ppc虚拟化平台的类首先通过`type_init()`注册到QOM系统中，在qemu其中时会通过回调函数的方式调用到虚拟化平台初始化函数，进而调用到cpu设备的实例化。调用栈如下：

```
machine_run_board_init() -> machine_class->init(machine)  // 回调函数初始化虚拟化平台
```

如果是e500plat的虚拟化平台将会调用到`ppce500_init()`，在`ppce500_init`就会通过`object_new()`创建cpu对象实例。调用过程大致如下

```
object_new() -> ... -> instance_init() -> qdev_realize_and_unref() -> qdev_realize() -> object_property_set_bool()
-> object_property_set -> prop->set() -> property_set_bool() -> device_set_realized()
```

`object_property_set_bool()`设置`realized`字段为true，最后会调用到`device_set_realized()`, `device_set_realized()`再通过回调函数调用到之前注册的函数，如`dc->realize() -> ppc_cpu_realize()`。

**这里可以发现通过通过回调函数`dc->realize()`真正调用到了具体cpu设备的实例化方法`ppc_cpu_realize()`，而这个方法正是在`translate_init.c.inc:ppc_cpu_class_init()`中通过调用`device_class_set_parent_realize()`注册到哈希表中的，而`ppc_cpu_class_init()`是通过`type_init()`注册到QOM系统中的。而`translate_init.c.inc`文件通过include的方式引入到target/ppc/translate.c文件从而加入编译的。**

在`ppc_cpu_realize()`中就会完成cpu实例化，通过`qemu_init_vcpu()`启动vcpu线程等操作完成一个cpu设备创建。

```
ppc_cpu_realize() {
	cpus.c:qemu_init_vcpu()
}
```



