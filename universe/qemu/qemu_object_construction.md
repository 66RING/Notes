---
title: QOM系统的"对象构建"子系统
author: 66RING
date: 2022-03-01
tags: 
- qemu
- QOM
mathjax: true
---

# Preface

关于为何需要使用`device_class_set_parent_realize()`的讨论

```
void device_class_set_parent_realize(DeviceClass *dc,
                                     DeviceRealize dev_realize,
                                     DeviceRealize *parent_realize)
{
    *parent_realize = dc->realize;
    dc->realize = dev_realize;
}
```

从逆向`device_class_set_parent_realize`的过程中探测到了一些QOM的设计哲学。


# QOM系统的"对象构造"子系统

**`device_class_set_parent_realize()`是让用户方便的构造"`realize`调用链"的API**。因为用户自己定义的设备有自己的realize方法，需要将realize方法加入到QOM系统中，使用`device_class_set_parent_realize()`就能方便的创建"继承关系的调用链"。

> Q: QOM不是实现了OOP的继承会自己调用父类的"构造"方法吗？为何还要手动构造调用链？
> 
> A: QOM的继承仅仅能实现`instance_init`的调用链，而不能调用父类的`realize`。这个根据`TypeInfo`的定义可知。而且`realize`是基于property系统(哈希表)的而不是基于`TypeInfo`。而QOM系统中一个设备的构造是大致分为了`object_initialize`, `instance_init`和`realize`三个部分的。所以需要手动构建`realize`调用链。至于为何QOM会将设备实例化过程分解为三个部分后面章节进行讨论。

```c
struct TypeInfo
{
    const char *name;
    const char *parent;

    size_t instance_size;
    size_t instance_align;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
```

对于任意设备的实例化方法`xxx_realize`，**基本都有如下的格式**：

```c
void xxx_realize() {
	...
	// 自身realize完成后 结尾调用parent_realize
	xxxClass->parent_realize();
}
```

因此就会形成向上调用的调用链，会依据这样的结构依次向上对父类进行实例化。 而对于`device_class_set_parent_realize()`

```
device_class_set_parent_realize(dc, ppc_cpu_realize, &pcc->parent_realize);

void device_class_set_parent_realize(DeviceClass *dc,
                                     DeviceRealize dev_realize,
                                     DeviceRealize *parent_realize)
{
    *parent_realize = dc->realize;
    dc->realize = dev_realize;
}
```

**总是会将`dc->realize`设备"叶子设备"的realize**。这样一来，**`dc->realize`就是设备realize过程的统一接口**。调用`dc->realize`后就会根据上述调用链调用父设备。

而为什么我觉得`dc->realize`总是叶子设备的realize方法并作为统一接口呢，一来是`device_class_set_parent_realize`的代码总是`dc->realize=dev_realize`，二来是注释说`TYPE_DEVICE`(即dc)是没有realize方法的，刚好可以留用作统一接口。

> Since TYPE_DEVICE doesn't implement @realize and @unrealize, types
> derived directly from it need not call their parent's @realize and
> @unrealize.
> For other types consult the documentation and implementation of the
> respective parent types.


## 几种"构造函数"的区别

- `object_initialize`
	* TODO 未知
- `instance_init`
- `realize`

根据`qdev-core.h`中注释的描述，我猜测，`instance_init`似乎只是设置了一些初始值，这点和我们在OOP语言中对象实例(instance)一创建就会调用构造函数不同。

> Trivial field initializations should go into #TypeInfo.instance_init.
> Operations depending on @props static properties should go into @realize.
> After successful realization, setting static properties will fail.

在QOM中对象真正的构造函数应该是`realize`。即根据构造参数(depending on properties)执行构造函数: "Operations **depending on @props static properties** should go into @realize."。*不过看具体的`realize`代码不是很能感受到这一点。e.g. `ppc_cpu_realize`*

QOM将对象实例化过程分成了`instance_init`和`realize`两个阶段，`instance_init`是在`TypeInfo`中描述的会在`object_new`时递归调用`object_new_with_type`, `object_init_with_type`等操作。而`realize`是基于property(哈希表)的，需要在必要时人工`realize`父设备(即有些设备虽然继承，但是是不需要`realize`的)。见如下注释的说明: 

> Any type may override the @realize and/or @unrealize callbacks but needs
> to call the parent type's implementation if keeping their functionality
> is desired. Refer to QOM documentation for further discussion and examples.

```
static void object_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti)) {
        object_init_with_type(obj, type_get_parent(ti));
    }
}
```

