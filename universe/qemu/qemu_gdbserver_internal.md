---
title: qemu gdbserver internel
date: 2021-08-21
tags: 
- qemu
- gdbserver
mathjax: true
---

# Preface

gdb设置断点的原理是使用`ptrace`系统调用建立gdb与目标程序的调试关系，gdb因此能够控制目标程序并先行对信号进行处理。因此设置断点的原理是：在断点处插入`int 3`触发软中断，gdb先行受到该软中断发出的SIGTRAP信号，从而转入gdb命令行，被调试程序停止。

利用gdb远程连接到和qemu中的gdbserver可以调试linux内核的执行过程。那qemu内的gdbserver是如何触发断点让程序停止的？如果也是使用`int 3`中断，那如果qemu已经是处在被gdb调试的状态(即`gdb qemu -S -s [...]`)则如何区分`int 3`是来自`gdb调qemu`的还是`qemu内gdbserver调内核`的？如果qemu内部`gdbserver`利用硬件来触发断点，那有些虚拟机不存在此类硬件又该如何处置？

下面将会对qemu内部gdbserver执行流程进行说明，它既不是利用int 3中断，也不是利用特殊的硬件支持，而是将断点target block翻译成"调用`helper_debuge`"退出cpu循环，再利用qemu的线程间通信(通过一些全局变量判断状态等)，通知到gdbserver的处理函数。

简单的说，`cpu_exec`执行到"断点代码"后停止，检测cpu退出原因，调用相应的异常/中断处理函数。如这里是触发断点，cpu退出后，在`switch case`到`EXCP_DEBUG`执行`cpu_handle_guest_debug`，修改`debug_requested`全局变量。主循环检测到`debug_requested`会触发`vm_stop`改变虚拟机状态。

```
r = tcg_cpu_exec(cpu);
switch (r) {
case EXCP_DEBUG:
	cpu_handle_guest_debug(cpu);
	break;
...
default:
	/* Ignore everything else? */
	break;
}
```


```
static bool main_loop_should_exit(void)
{
	...
    if (qemu_debug_requested()) { 		// 检测到debug请求，vm stop
        vm_stop(RUN_STATE_DEBUG);
    }
	...
    if (qemu_suspend_requested()) {
        qemu_system_suspend();
    }
	...
}
```


# Internal

本质上qemu中的gdbserver也在目标程序(被调试OS)插入"断点代码"，不同的是它不是根据`int 3`找到中断处理程序，而是利用cpu循环检测退出原因后修改一些标记(`debug_requested`)从而主线程能探测到，然后执行"中断处理函数"。

当使用`-S -s`参数(虚拟机停止，并开启默认gdbserver设置)启动虚拟机，在虚拟机初始化时会调用`gdbserver_start()`将`gdb_vm_state_change()`加入到虚拟机状态变化后需要执行的handler队列中，这样虚拟机状态改变(如`vm_stop`)就会执行`gdb_vm_state_change`。

```c
qemu_add_vm_change_state_handler(gdb_vm_state_change, NULL);
```

TODO: `gdb_vm_state_change`功能概括

虚拟机初始化完毕后，cpu处于停止状态，即`!cpu_can_run()`，因此不会进入`cpu_exec`循环。cpu处在`qemu_wait_io_event()`的等待中，远程连接的gdb(以下称client)与qemu内的gdbserver(以下称server)的连接建立后，client和server的通信不需要经过`cpu_exec`的处理。

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/qemu_gdbserver_internal/communicate_without_cpu.png" alt="" width="100%">

Q?? 没能确定是哪个线程处理client和server通信，然后触发dispatch，主循环如何poll?还是系统内?

client中维护调试信息，而server仅负责获取client需要的信息并发送。如当client设置断点时(即`break bp`操作)，通过追踪`gdb_handle_packet`的执行发现server的处理函数是`read_mem_cmd_desc`，server并没有直接插入断点。

一个合理的解释是：插入断点需要重新生成tb(以tcg为例)，而程序在`continue`之前可能重复的插入/删除断点，因此在必要时机(如continue后)统一，就节省了tb的生成/删除的开销。

当client输入`continue`命令后，将积累的信息发送给server，server进行处理。这是才会真正插入断点，因为cpu马上就要运行了。server发现断点会通过`cpu_breakpoint_insert()`将断点加入cpu的断点列表`cpu->breakpoints`，并执行`breakpoint_invalidate() -> tb_flush()`，将`do_tb_flush`放入cpu的工作队列(work item list)，并用`qemu_cond_broadcast()`通知在`qemu_wait_io_event()`中等待的cpu有work item需要执行。之后恢复cpu的执行，即可以进入`cpu_exec`循环。

`do_tb_flush`强制刷新Target Block(以下称tb)的缓存信息，为后面翻译tb插入"断点代码"做准备。

```
int cpu_breakpoint_insert(CPUState *cpu, vaddr pc, int flags, CPUBreakpoint **breakpoint)
{
    CPUBreakpoint *bp;
    bp = g_malloc(sizeof(*bp));
    bp->pc = pc;
    bp->flags = flags;
    /* keep all GDB-injected breakpoints in front */
    if (flags & BP_GDB) {
        QTAILQ_INSERT_HEAD(&cpu->breakpoints, bp, entry);
    } else {
        QTAILQ_INSERT_TAIL(&cpu->breakpoints, bp, entry);
    }

    breakpoint_invalidate(cpu, pc);

    if (breakpoint) {
        *breakpoint = bp;
    }
    return 0;
}
```

`cpu_exec`从`qemu_wait_io_event()`中唤醒，先执行work item list中的任务，其中就包括刚刚传入的`do_tb_flush`清空tb缓存。之后就是`cpu_exec`循环。由于此时tb缓存为空，需要重新生成tb，在生成tb`translator_loop()`中就会根据`cpu->breakpoints`列表翻译代码块，然后插入"断点代码"。之后虚拟机正常运行事件循环。

以x86为例，当cpu执行到断点处时"断点代码"被tcg翻译成"执行`helper_debug`函数"，`helper_debug`触发cpu退出从而达到在断点处停止的效果。

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/qemu_gdbserver_internal/helper_debug.png" alt="" width="100%">

cpu退出后检查退出原因并执行相应的处理，这里就是调用`breakpoint_handler`：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/qemu_gdbserver_internal/handle_exception.png" alt="" width="100%">

cpu停止，虚拟机状态改变事件触发`vm_state_notify`，执行状态改变handler队列中的函数。gdbserver开始他的工作`gdb_vm_state_change`，根据状态生成信息通过`put_packet(buf->str);`(TODO进一步确认`put_packet`功能)通知到gdbserver"设备"，gdbserver接收到命令后，通过`cpu_breakpoint_remove()`移除断点。等待下次循环发生。

TODO: `put_packet()`是统一的接口，给到client端用(client命令到来-> 中间层解析 -> `put_packet`给server)，同时也给到qemu内部通信用。因为我如果在`gdb_vm_state_change`中跳过`put_packet()`的执行，就不会触发`cpu_breakpoint_remove`了。

TODO那如果 `put_packet`也是给gdbserver发命令了，又是那个接口负责回复client?

TODO
```
https://sourceware.org/gdb/onlinedocs/gdb/Remote-Protocol.html
即可能是有个中间层
client/qemu   -> 中间层  -`put_packet`-> gdbserver
```
TODO

完整调试流程如下：

TODO

```
b gdb_vm_state_change
// 当pc等于断点时触发
b cpu_loop_exec_tb if tb->pc == 0xffffffff8304ce5f
b cpu_breakpoint_insert
b cpu_breakpoint_remove
// 当pc等于断点时触发
b cpu_handle_interrupt if 0xffffffff8304ce5f == *&((CPUX86State *)&(X86_CPU(current_cpu)->env))->eip
// 监听breakpoint列表使用情况
rwatch &current_cpu->breakpoints->tqh_first if 0xffffffff8304ce5f == *&((CPUX86State *)&(X86_CPU(current_cpu)->env))->eip
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/qemu_gdbserver_internal/bp_demo.png" alt="" width="100%">



# monitor和plugin的区别

- plugin在cpu循环中，monitor在主循环中
- 预设点的plugin可以由很多方法触发，monitor就一种

一些位置预设有plugin，monitor可以控制状态改变从而达到控制plugin的效果

```
qemu_wait_io_event  	->   qemu_plugin_vcpu_resume_cb      // 如resume触发
~/var/QEMU_project/Qemu_p2020/qemu-5.2.0/softmmu/cpus.c

do_tb_flush 		-> 	 qemu_plugin_flush_cb
~/var/QEMU_project/Qemu_p2020/qemu-5.2.0/accel/tcg/translate-all.c

// plugin.h 中{}，但./plugins/core.c中有具体实现，一般跳到plugin.h的空，而不是具体实现
```

`~/var/QEMU_project/Qemu_p2020/qemu-5.2.0/include/qemu/plugin.h`中似乎存着所有接口


# monitor还能干嘛


