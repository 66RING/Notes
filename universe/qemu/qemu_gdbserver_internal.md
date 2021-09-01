---
title: qemu gdbserver internal
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

当使用`-S -s`参数(虚拟机停止，并开启默认gdbserver设置)启动虚拟机，在虚拟机初始化时会调用`gdbserver_start()`将`gdb_vm_state_change()`加入到虚拟机状态变化后需要执行的handler队列中，这样虚拟机状态改变(如`vm_stop`)就会执行`gdb_vm_state_change`。`gdb_vm_state_change`作为入口switch case到相应的流程。

```c
qemu_add_vm_change_state_handler(gdb_vm_state_change, NULL);
```

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

TODO: 这里猜测`put_packet()`是统一的接口，给到client端用(client命令到来-> 中间层解析 -> `put_packet`给server)，同时也给到qemu内部通信用。因为我如果在`gdb_vm_state_change`中跳过`put_packet()`的执行，就不会触发`cpu_breakpoint_remove`了。

完整调试流程如下：

```
b gdb_vm_state_change
// 当pc等于断点时触发
b cpu_loop_exec_tb if tb->pc == 0xffffffff8304ce5f
b cpu_breakpoint_insert
b cpu_breakpoint_remove
tb cpu_exec
commands
silent
p /a &((CPUX86State *)&(X86_CPU(current_cpu)->env))->eip
set $IP_ADDR=$
// 当pc等于断点时触发
b cpu_handle_interrupt if 0xffffffff8304ce5f == *$IP_ADDR
// 监听breakpoint列表使用情况
p &current_cpu->breakpoints->tqh_first
set $BP=$
rwatch *$BP if 0xffffffff8304ce5f == *$IP_ADDR
p &current_cpu->exception_index
set $EXCP=$
awatch *$EXCP if *$==0x10002
end
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/qemu_gdbserver_internal/bp_demo.png" alt="" width="100%">


## KVM

上面介绍的流程是基于tcg的，在启用kvm的情况下cpu循环退出的方式稍微有点不同，但事件循环模型是差不多的。

开启kvm后虚拟机cpu的执行可以在一个特殊的模式下进行(guest mode)，这个模式下执行的虚拟机就相当于在host中运行的一个普通进程，kvm和系统调度会为何进程上下文的切换。依赖于具体的指令集体系结构，kvm执行到的host cpu无法处理的指令(如x86检测到debug寄存器断点触发、设备模拟等)后会导致vmexit退出guest mode进入qemu模拟。

TODO: 整个vmexit过程可能依赖于具体ISA和硬件。如检测debug寄存器dr7等?这里具体哪里触发vmexit没有搞清楚。需要进一步了解具体如何exit的，如何执行时知道要去qemu模拟了，替换了一些指令?? [may help](https://gist.github.com/mcastelino/b31f0648707b25478eb2a44f94a861fd)

### KVM下的执行流程: 从断点注入到vmexit

断点信息从client发送到server后，gdbserver解析到断点内容，使用`ioctl`+`FLAGS`的方式设置好断点。调用流程toplevel view如下

```
gdb_handle_packet -> ... -> kvm_invoke_set_guest_debug -> kvm_vcpu_ioctl 

-> ioctl(kvm_fd, KVM_SET_GUEST_DEBUG, &dbg_data->dbg) 
```

qemu中传入FLAGS和相应的参数`dbg_data->dbg`，kvm中通过判断FLAGS进入相应的处理：

```
static long kvm_vcpu_ioctl(struct file *filp,
			   unsigned int ioctl, unsigned long arg)
{
		/* ... */
	switch (ioctl) {
	case KVM_RUN: {}
	case KVM_GET_REGS: {}
	case KVM_SET_REGS: {}
	case KVM_GET_SREGS: {}
		/* ... */
	case KVM_SET_GUEST_DEBUG: kvm_arch_vcpu_ioctl_set_guest_debug();
		/* ... */
}
```

在`kvm_arch_vcpu_ioctl_set_guest_debug`中可以看到，对于x86架构，会使用专门的debug register来实现断点注入操作，作为后面进入guest mode后指令执行。具体x86中的这个流程是如何进行的先按下不表。

```
int kvm_arch_vcpu_ioctl_set_guest_debug(struct kvm_vcpu *vcpu,
					struct kvm_guest_debug *dbg)
{
	/* ... */
	vcpu_load(vcpu);

	vcpu->guest_debug = dbg->control;
	if (!(vcpu->guest_debug & KVM_GUESTDBG_ENABLE))
		vcpu->guest_debug = 0;

	/* ... */
	if (vcpu->guest_debug & KVM_GUESTDBG_USE_HW_BP) {
		for (i = 0; i < KVM_NR_DB_REGS; ++i)
			vcpu->arch.eff_db[i] = dbg->arch.debugreg[i];
		vcpu->arch.guest_debug_dr7 = dbg->arch.debugreg[7];
	} else {
		for (i = 0; i < KVM_NR_DB_REGS; i++)
			vcpu->arch.eff_db[i] = vcpu->arch.db[i];
	}
	kvm_update_dr7(vcpu);
	/* ... */

	static_call(kvm_x86_update_exception_bitmap)(vcpu);
	/* ... */
	return r;
}
```

可见kvm中将debug寄存器中的信息保存到了vcpu中，为后面vcpu的执行提供依据。

当之后gdbserver会让虚拟机进入执行状态，即qemu进入vcpu执行循环。同样通过`ioctl`调用kvm：

```
gdb_handle_packet -> ... -> qemu_cpu_kick -> ... -> qemu_cond_broadcast ->
-> kvm_cpu_exec
```

`kvm_cpu_exec`cpu执行后就会调用`ioctl(cpu, KVM_RUN)`启动kvm进入guest，kvm就会进入`KVM_RUN`的case：

```
kvm_vcpu_ioctl -> kvm_arch_vcpu_ioctl_run -> ... -> vmx_vcpu_run -> vmx_vcpu_enter_exit
```

```
vcpu_enter_guest(){
	/* 准备进入guest mode ... */
	vcpu->mode = IN_GUEST_MODE;
	for (;;) {
		/* ... */
		exit_fastpath = static_call(kvm_x86_run)(vcpu);  // vmx_vcpu_run
		/* ... */
	}
	/* 退出guest mode ... */
	vcpu->mode = OUTSIDE_GUEST_MODE;
	r = static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath);
	return r;
}
```

经过一系列准备后(如判断vcpu标记位、保存host信息、载入guest信息等)后`vmx_vcpu_enter_exit`调用`__vmx_vcpu_run`进入guest mode。`__vmx_vcpu_run`汇编代码可见，进入会保存host寄存器现场然后载入guest寄存器。这样一来虚拟机就相当于host上的一个普通进程调度运行，当kvm执行到vmexit后会从kvm退出到qemu进行模拟/处理。guest mode退出后又会保存guest恢复host。

```
SYM_FUNC_START(__vmx_vcpu_run)
	/* 保存host的寄存器 */
	push %_ASM_BP
	mov  %_ASM_SP, %_ASM_BP
	push %r15
	/* ... */

	/* 准备guest的寄存器 */
	mov VCPU_RCX(%_ASM_AX), %_ASM_CX
	/* ... */

	/* 进入guest mode */
	call vmx_vmenter

	/* 如果vm_fail则跳到vm_fail处理另返回1，否则0 */
	jbe 2f

	/* 善后工作保存guest的寄存器等 */

	/* 清空通用寄存器 */
1:	xor %ecx, %ecx
	xor %edx, %edx
	xor %ebx, %ebx
	/* 恢复host现场 */
	/* ... */
	ret

	/* VM-Fail 返回1 */
2:	mov $1, %eax
	jmp 1b
SYM_FUNC_END(__vmx_vcpu_run)
```

qemu会处理kvm的退出原因后做相应的处理，如这里坚持到断点触发后qemu设置特殊标记，主循环检测到标记后令cpu循环停止，并调用虚拟机初始化时注册的handler(如这里是`gdb_vm_state_change`)做相应处理。从而实现gdbserver断点触发的停止效果。

```
static void *kvm_vcpu_thread_fn(void *arg)
{
	/* ... */
	r = kvm_cpu_exec(cpu);
	if (r == EXCP_DEBUG) {
		cpu_handle_guest_debug(cpu); // call qemu_system_debug_request
	}
	/* ... */
}
```

```
main_loop_should_exit -> vm_stop -> vm_state_notify -> gdb_vm_state_change
```

这时cpu停止，进入下一个gdbserver响应client命令的循环。

