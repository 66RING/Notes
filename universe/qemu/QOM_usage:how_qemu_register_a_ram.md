---
title: QOM使用：qemu如何注册使用一个MemoryRegion类的
date: 2021-04-17
tags: 
- qemu
- qom
mathjax: true
---

# openrisc中ram对象的生成过程

QEMU利用QOM来对对象进行抽象，用它来对各种资源进程抽象、管理(创建、配置、销毁)。即QEMU内部实现面向对象机制的方法。

QOM机制有如下三个部分：

- 类型注册，涉及函数有`type_init`、`register_module_init`、`type_register`
    * 生成`TypeImpl`，通过`TypeImpl`就可以知道如何初始化一个类
    * 相当于告诉qemu这个类如何定义，如:
        + 类名是什么：`.name`
        + 类实例化调用什么函数：`.instance_init`
        + 类自己的特有方法、对父类如何继承等：`.class_init`
- 类型的初始化`type_initialize`
    * 类型定义告诉了QEMU怎么初始化类，类型初始化就根据定义的内容把类创建出来
- 对象的初始化
    * 通过上面创建好的类来实例化对象

MemoryRegion也是一个QOM对象，在`./softmmu/memory.c`进行了类型注册：

```
static const TypeInfo memory_region_info = {
    .parent             = TYPE_OBJECT,
    .name               = TYPE_MEMORY_REGION,
    .class_size         = sizeof(MemoryRegionClass),
    .instance_size      = sizeof(MemoryRegion),
    .instance_init      = memory_region_initfn,
    .instance_finalize  = memory_region_finalize,
};

type_register_static(&memory_region_info);
```

其中类名`TYPE_MEMORY_REGION`展开为`qemu:memory-region`，每个MemoryRegion类的实例通过`memory_region_initfn`构造，通过`memory_region_finalize`销毁。通过以上信息QEMU就能知道如何创建MemoryRegion类，以及MemoryRegion类实例该如何生成。

MemoryRegion可以分为如下几类

- 根级MemoryRegion
    * 没有自己的内存，用于管理subregion，如`system_memory`
    * 通过`memory_region_init`初始化
- **实体MemoryRegion**
    * **有具体的内存**，从QEMU进程地址空间分配内存
    * 通过`memory_region_init_ram`初始化，实际由`qemu_ram_alloc`分配实际内存。如RAM、ROM、ROM device等
- 别名MemoryRegion
    * 表示实体mr的不同分段，没有自己的内存，是实体mr的一部分
    * 通过`memory_region_init_alias`初始化
    * 通过`alias`成员 **指向实体MemoryRegion** ，`alias_offset`表示该别名mr在实体内存中的偏移量

ram作为根级MemoryRegion，也是MemoryRegion类的一个实例，通过`memory_region_init_ram`创建和初始化。

openrisc中对其ram的构造方法如下：

```
// 函数原型
void memory_region_init_ram(MemoryRegion *mr, struct Object *owner, const char *name, uint64_t size, Error **errp)

memory_region_init_ram(ram, NULL, "openrisc.ram", ram_size, &error_fatal);
```

可见对象名将要设置为`openrisc.ram`，`memory_region_init_ram`的调用关系为：

```
memory_region_init_ram()
    memory_region_init_ram_nomigrate()
        memory_region_init_ram_shared_nomigrate()
            memory_region_init(mr, owner, name, size);
```

`memory_region_init_ram`其底层也是调用`memory_region_init`做通用MemoryRegion的初始化，只不过在上层，如`memory_region_init_ram_shared_nomigrate()`中会针对ram做一些定制的操作。

`memory_region_init()`主要有两部

```c
void memory_region_init(MemoryRegion *mr,
                        Object *owner,
                        const char *name,
                        uint64_t size)
{
    object_initialize(mr, sizeof(*mr), TYPE_MEMORY_REGION);
    memory_region_do_init(mr, owner, name, size);
}
```

- `object_initialize(mr, sizeof(*mr), TYPE_MEMORY_REGION);`
    * 为ram生成MemoryRegion类对象/实例
- `memory_region_do_init(mr, owner, name, size);`
    * 对实例化后的MemoryRegion对象进行初始化


## object_initialize

`object_initialize()`会通过`object_initialize_with_type()`根据传入的typename符串找到类的描述，从而初始化一个QOM对象。

这里typename为`TYPE_MEMORY_REGION`宏，展开为`qemu:memory-region`，也就是MemoryRegion类注册的类名。通过类名找到了MemoryRegion类，用于做MemoryRegion类的实例化。这里是对`openrisc.ram`实例化

```c
void object_initialize(void *data, size_t size, const char *typename)
{
    TypeImpl *type = type_get_by_name(typename);    // 根据类名通过hash表找到对应的TypeImpl结构体，QOM用TypeImpl来描述一个类的信息
    // 注册了但未必初始化了，故后面还有确保初始化了的操作
    ...
    object_initialize_with_type(data, size, type);  // 根据类描述结构(TypeImpl)初始化对象
}
```

`object_initialize_with_type()`内容如下：

```c
static void object_initialize_with_type(Object *obj, size_t size, TypeImpl *type)
{
    type_initialize(type); // 确保类已经初始化过了，如果已经初始化则跳过

    g_assert(type->instance_size >= sizeof(Object));
    g_assert(type->abstract == false);
    g_assert(size >= type->instance_size);

    memset(obj, 0, type->instance_size); // 卡号四初始化，这里传入obj是mr
    obj->class = type->class;
    object_ref(obj);            // 增加基类引用计数
    object_class_property_init_all(obj);
    // 为该对象生成property域，其属性就可以通过hash表获取
    obj->properties = g_hash_table_new_full(g_str_hash, g_str_equal,
                                            NULL, object_property_free);
    object_init_with_type(obj, type);       // 调用自身的初始化函数和递归调用所有父类的初始化函数
    object_post_init_with_type(obj, type);  // 调用TypeImpl的instance_post_init回调函数，完成对象初始化之后的工作
}
```

`object_init_with_type`会调用自身的实例化函数并递归地调用父类的实例化函数。对于`MemoryRegion`类的对象，其实例化函数为`memory_region_initfn()`

```c
static void memory_region_initfn(Object *obj)
{
    MemoryRegion *mr = MEMORY_REGION(obj);
    ObjectProperty *op;

    mr->ops = &unassigned_mem_ops;
    mr->enabled = true;
    mr->romd_mode = true;
    mr->destructor = memory_region_destructor_none;
    QTAILQ_INIT(&mr->subregions);
    QTAILQ_INIT(&mr->coalesced);

    op = object_property_add(OBJECT(mr), "container",
                             "link<" TYPE_MEMORY_REGION ">",
                             memory_region_get_container,
                             NULL, /* memory_region_set_container */
                             NULL, NULL);
    op->resolve = memory_region_resolve_container;

    object_property_add_uint64_ptr(OBJECT(mr), "addr",
                                   &mr->addr, OBJ_PROP_FLAG_READ);
    object_property_add(OBJECT(mr), "priority", "uint32",
                        memory_region_get_priority,
                        NULL, /* memory_region_set_priority */
                        NULL, NULL);
    object_property_add(OBJECT(mr), "size", "uint64",
                        memory_region_get_size,
                        NULL, /* memory_region_set_size, */
                        NULL, NULL);
}
```

`memory_region_initfn`会给该MemoryRegion对象做如下基本设置：

- 赋值默认操作函数`unassigned_mem_ops`
- 添加link类型的container属性
    * 每个mr都可以是其他mr的container
    * link属性表示一种连接关系，表示一种设备引用另一种设备
- 添加priority属性
- 添加size属性


## memory_region_do_init

通过上面的步骤就生成了一个MemoryRegion实例，通过`memory_region_do_init`再对该MemoryRegion实例进程初始化(本例是是openrisc.ram的初始化，以下就称为openrisc.ram对象或openrisc.ram实例)

`memory_region_do_init`对实例初始化，如设置名称、设置owner执行父对象、添加属性等：

```c
static void memory_region_do_init(MemoryRegion *mr,
                                  Object *owner,
                                  const char *name,
                                  uint64_t size)
{
    mr->size = int128_make64(size);
    if (size == UINT64_MAX) {
        mr->size = int128_2_64();
    }
    mr->name = g_strdup(name);
    mr->owner = owner;
    mr->ram_block = NULL;

    if (name) {
        char *escaped_name = memory_region_escape_name(name);   // 去转意??
        char *name_array = g_strdup_printf("%s[*]", escaped_name);

        if (!owner) {
            owner = container_get(qdev_get_machine(), "/unattached");
        }

        object_property_add_child(owner, name_array, OBJECT(mr));
        object_unref(OBJECT(mr));
        g_free(name_array);
        g_free(escaped_name);
    }
}
```

openrisc.ram的初始化过程传入的owner为NULL，这样就会分配一个叫"/unattached"的Object作为其owner，然后将openrisc.ram实例作为owner的child属性：

然后通过qemu monitor查看qom信息:`info qom-tree`。就会发现openrisc.ram对象是`unattached`对象的一个child属性

![openrisc_qom_tree](https://raw.githubusercontent.com/66RING/66RING/master/.github/Notes/universe/qemu/openrisc_qom_tree.png)

todo 对象之间怎么联系的？


# 抽象

## 接口

### 类注册

使用TypeInfo结构来描述一个类，其中一些个重要成员如下：

- `parent`，父类名
- `name`，该类型的类名
- `class_size`，该类的大小
- `class_init`，该类的初始化函数
- `instance_size`，该类的实例对象的大小
- `instance_init`，该类的实例的实例化函数
- `instance_finalize`，该类实例的销毁函数

```
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

定义好一个类后需要使用`type_register_static(YOUR_TYPEINFO);`将类注册到全局hash表，用于根据类名构造类和类实例。

`type_register_static`可以将类信息注册，但一般不直接调用或者用到时在调用。而是使用`type_init(YOUR_REGISTER)`让所有类注册在main之前完成。`type_init`是个宏，其又是`module_init`宏，`module_init`宏如下：

```c
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
```

`module_init`宏通过添加编译器属性`__attribute__((constructor))`保证注册函数会在`main`函数前调用。

类似的宏还有`block_init`、`opts_init`、`trace_init`等


### 类初始化

通过定义`TypeInfo`和注册，仅仅是告诉了qemu要怎么构造类，要生成类的实例还要调用`type_initialize`将类创建出来。即类的初始化通过`type_initialize`完成。

其先判断`ti->class`如果非空说明类已经初始化过了。因此使用`object_initialize_with_type`其实是类对象的一种懒加载方法。

```c
static void object_initialize_with_type(Object *obj, size_t size, TypeImpl *type)
{
    type_initialize(type);
    ...
}
```

其如果是第一次执行则会进入`type_initialize`，发现

不过大部份类初始化在创建虚拟机时就通过下面步骤初始化好了。所以`object_initialize_with_type`的`type_initialize`更多还是在保证类确实已经创建了。没有创建再调用创建。

```
qemu_init
    select_machine
        object_class_get_list
            object_class_foreach
                g_hash_table_foreach
                    object_class_foreach_tramp
                        type_initialize
```


### 通过类实例化一个对象

```c
void object_initialize(void *data, size_t size, const char *typename)
```

- data用于接受要实例化的对象
    * 如`mr`
- size表示该实例的大小
    * 如`sizeof(*mr)`
- typename表示类名

其内部有的核心调用为：

```c
|- TypeImpl *type = type_get_by_name(typename);
|- object_initialize_with_type(type)
    |- ...
    |- type_initialize(type);
    |- obj->class = type->class;
    |- object_class_property_init_all(obj);
    |- obj->properties = g_hash_table_new_full(g_str_hash, g_str_equal,
    |-                                         NULL, object_property_free);
    |- ...
    |- object_init_with_type(obj, type);
    |- object_post_init_with_type(obj, type);
```

`object_initialize`会根据类名找到对应的类，然后通过`object_initialize_with_type`实例化，其中实例化的核心是`object_init_with_type(obj, type)`。它会根据类的实例化函数`instance_init`将对象实例化，并递归的实例化父类。

而`object_post_init_with_type`是处理对象实例化后的一些通过，会根据类的`instance_post_init`函数完成。


### 初始化对象细节

像MemoryRegion类，其对象除了基础的实例化还有一些细节需要初始化。这就要为什么`memory_region_init()`是这样的：

```c
void memory_region_init(MemoryRegion *mr,
                        Object *owner,
                        const char *name,
                        uint64_t size)
{
    object_initialize(mr, sizeof(*mr), TYPE_MEMORY_REGION);
    memory_region_do_init(mr, owner, name, size);
}
```

模仿MemoryRegion对象的构建方法，用户可以对自己的对象构造方法进一步封装分别实例化对象和补充细节。对应于`memory_region_init`中的的`object_initialize`和`memory_region_do_init`

以MemoryRegion对象为例，其`memory_region_do_init`为MemoryRegion对象增加了`owner`、`name`域，还将对象本身添加到了父对象的child属性中。


## 属性

为了对对象进行管理，每种类型的对象都增加了 **属性**。类属性在`ObjectClass`(所有QOM对象的父类)的`properties`域中。

```c
struct ObjectProperty
{
    char *name;     // 属性名
    char *type;     // 属性类型bool、link或更复杂的类型
    char *description;  // 表述
    ObjectPropertyAccessor *get;  // 操作属性的一系列回调函数
    ObjectPropertyAccessor *set;
    ObjectPropertyResolve *resolve;
    ObjectPropertyRelease *release;
    ObjectPropertyInit *init;
    void *opaque;       // 指向一个具体的属性，如LinkProperty、BoolProperty等
    QObject *defval;
};
```

每种具体的属性都有一个结构体描述它。如`LinkProperty`结构体表示link类型的属性，`StringProperty`表示字符串类型的属性，`BoolProperty`表示bool类型的属性。

`LinkProperty`属性有个`Object **child`成员，用于连接两个对象


### 设备间通过属性交互

- link/child属性
    * link属性表示一种连接关系，表示一种设备引用另一种设备
    * child属性表示对象之间的从属关系。对象的child属性执行子对象


todo 添加属性怎么添加的，怎么实现的，`object_property_add`















