---
title: qemu中的初始化技术
date: 2021-04-12
tags: 
- qemu
- c
- gcc
mathjax: true
---

https://blog.csdn.net/u011364612/article/details/53581501

- todo
    * 全局`MemoryRegion`创建时机
    * 虚拟机创建时机
    * qom创建时机


初始化函数会将

qemu代码的初始化是分模块管理的。有如下4中模块：

- block（QEMU中的块操作实现代码）
- machine（QEMU中模拟设备的实现代码模块）
- qapi（QEMU向上层调用者提供的接口的代码模块）
- type或者qom（QEMU中的QOM模型所涉及的代码模块）

```c
typedef enum {
    MODULE_INIT_BLOCK,
    MODULE_INIT_MACHINE,
    MODULE_INIT_QAPI,
    MODULE_INIT_QOM,
    MODULE_INIT_MAX
} module_init_type;

#define block_init(function) module_init(function, MODULE_INIT_BLOCK)
#define machine_init(function) module_init(function, MODULE_INIT_MACHINE)
#define qapi_init(function) module_init(function, MODULE_INIT_QAPI)
#define type_init(function) module_init(function, MODULE_INIT_QOM)
```

QEMU中使用qom模型设计的类需要注册到全局的hash表中。将初始化函数保存到链表中。这样只需调用链表中每个entry对应的初始化函数即可实现初始化工作。

> gcc中，使用`__attribute__((constructor))`属性使函数在main函数执行之前执行，同理使用`__attribute__((destructor))`属性让其在main之后执行。

通过`register_module_init`函数将初始化函数存入全局的`init_type_list`的对应模块的链表中。在由宏展开自动生成构造函数。

如`type_init(register_types)`会自动生成如下函数：

```c
static void __attribute__((constructor)) do_qemu_init_register_type(void)    
{                                                                           
    register_module_init(register_type, MODULE_INIT_QOM);                                   
}
```

当执行到初始化machine时


## doc

分模块，链表，简化初始化

```c
type_init(register_gtk);
```

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

#define block_init(function) module_init(function, MODULE_INIT_BLOCK)
#define opts_init(function) module_init(function, MODULE_INIT_OPTS)
#define type_init(function) module_init(function, MODULE_INIT_QOM)
#define trace_init(function) module_init(function, MODULE_INIT_TRACE)
#define xen_backend_init(function) module_init(function, \
                                               MODULE_INIT_XEN_BACKEND)
#define libqos_init(function) module_init(function, MODULE_INIT_LIBQOS)
#define fuzz_target_init(function) module_init(function, \
                                               MODULE_INIT_FUZZ_TARGET)
#define migration_init(function) module_init(function, MODULE_INIT_MIGRATION)
```

vvv`module_init`vvv

```c
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_dso_module_init(function, type);                               \
}
#else
/* This should not be used directly.  Use block_init etc. instead.  */
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
#endif
```

所以对`type_init()`的调用最终会然它宏展开为编译器属性包裹的函数`__attribute__((constructor)) do_qemu_init_ ## function(void)`使其能在main之前执行。

插入对应类型的....

```c
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

`qemu_inti()`初始化时调用`module_call_init()`完成给定类型的模块的初始化。

```c
void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    if (modules_init_done[type]) {
        return;
    }

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {
        e->init();
    }

    modules_init_done[type] = true;
}
```






