---
title: qemu中的初始化技术
date: 2021-04-12
tags: 
- qemu
- c
- gcc
mathjax: true
---

# qemu的初始化框架

qemu启动需要初始化很多内容，为了方便维护和添加初始化函数，qemu有它自己的一种实现方式。

qemu的初始化时根据具体的模块类型，执行该类型的所有模块的初始化。有如下类型：

```c
typedef enum {
    MODULE_INIT_MIGRATION,
    MODULE_INIT_BLOCK,
    MODULE_INIT_OPTS,
    MODULE_INIT_QOM,
    MODULE_INIT_TRACE,
    MODULE_INIT_XEN_BACKEND,
    MODULE_INIT_LIBQOS,
    MODULE_INIT_FUZZ_TARGET,
    MODULE_INIT_MAX
} module_init_type;
```

有一个全局的模块类型数组`init_type_list`，数组中每项对应一种类型的模块。每个模块有自己的初始化函数，模块间用`ModuleTypeList`类型的链表组织，初始化就可以针对一种类型的模块，遍历其链表来完成该类型的所有模块的初始化：

```c
static ModuleTypeList init_type_list[MODULE_INIT_MAX];
```

qemu初始化框架的核心其实是使用编译器参数`__attribute__((constructor))`，使得模块注册函数在main函数之前执行。不同类型模块的注册函数接口不同，但本质都是对`register_module_init(function, type)`的封装，它会根据模块类型`type`，将初始化函数`function`添加到模块初始化链表末尾。

以`type_init`为例，它是QOM类型模块的注册接口函数，其他类型的宏定义同理：

```c
#define type_init(function) module_init(function, MODULE_INIT_QOM)
#define block_init(function) module_init(function, MODULE_INIT_BLOCK)
#define opts_init(function) module_init(function, MODULE_INIT_OPTS)
#define trace_init(function) module_init(function, MODULE_INIT_TRACE)
#define xen_backend_init(function) module_init(function, \
                                               MODULE_INIT_XEN_BACKEND)
#define libqos_init(function) module_init(function, MODULE_INIT_LIBQOS)
#define fuzz_target_init(function) module_init(function, \
                                               MODULE_INIT_FUZZ_TARGET)
#define migration_init(function) module_init(function, MODULE_INIT_MIGRATION)
```

`type_init`向`module_init`宏传入的type为`MODULE_INIT_QOM`。根据这个枚举值就能在全局的`init_type_list`数组中找到对应类型的模块链表。然后调用`module_init`将对应的初始化函数，即参数`function`，插入链表末尾：

```c
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_dso_module_init(function, type);                               \
}


void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);  // 返回对应类型模块的链表init_type_list[type]

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

`module_init`会展开成一系列`__attribute__((constructor)) do_qemu_init_ ## function(void)`函数。编译器参数`__attribute__((constructor))`保证了这些初始化函数在main函数前执行，即完成模块注册。

这样一来在qemu主初始化函数`qemu_init`中就可以在根据需要调用`module_call_init(type)`执行对应类型模块的初始化了：

```c
void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    if (modules_init_done[type]) {
        return;
    }

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {  // 执行注册好的初始化函数
        e->init();
    }

    modules_init_done[type] = true;
}
```


# 一个实例

以`memory_region_info`这个QOM对象为例，可以在源码`memory.c`中找到：

```c
static const TypeInfo memory_region_info = {
    ...
};

static const TypeInfo iommu_memory_region_info = {
    ...
};

static void memory_register_types(void)
{
    type_register_static(memory_region_info);
    ...
}

type_init(memory_register_types)
```

QOM对象调用`type_register_static()`，根据`TypeInfo`结构完成类的创建。那么就需要调用`type_init()`将类创建函数注册到模块链表中，确保初始化时创建好类。

`type_init`通过宏展开最终将`memory_register_types`展开成`do_qemu_init_memory_register_types`这个在main函数执行前调用的函数。它再通过`register_module_init`将`memory_register_types`插入`init_type_list[MODULE_INIT_QOM]`链表中。

这样QOM模块需要初始化时`qemu_init`就能通过`module_call_init(MODULE_INIT_QOM)`执行QOM类型模块的初始化函数，其中就包括刚才注册的`memory_register_types()`

