---
title: qemu monitor的工作模型
date: 2021-08-21
tags: 
- qemu
mathjax: true
---

# 前言

gdb调试程序会阻塞被调试程序运行，gdb走一步，程序才能走一步，形如：

```
gdb
proc
gdb
proc
```

那qemu的monitor中使用HMP/QMP与虚拟机交互交互会不会导致qemu挂起呢？monitor和qemu交互是同步还是异步？如果是同步那qemu是如何挂起的

monitor等待命令输入时不会阻塞qemu进行，不像gdb那样等待命令时阻塞被调试程序运行。只是在输入命令敲入回车的瞬间，vCPU将事件通知到主循环让主循环dispatch执行，而在这之前主循环一直在处理其他工作，主循环执行完成后vCPU继续他的工作。


# 存在几种同步方法

可以用work item和全局锁同步。测试方法:`info register`

主循环循环prepare，poll，dispatch等待vCPU通知事件发生。当与monitor交互后主循环dispatch到monitor的处理流程：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_monitor_model/dispatch_monitor.png" alt="">

单步跟踪发现`info register`对应的处理函数为`cpu_dump_state`：

```c
void cpu_dump_state(CPUState *cpu, FILE *f, int flags)
{
    CPUClass *cc = CPU_GET_CLASS(cpu);

    if (cc->dump_state) {
        cpu_synchronize_state(cpu);
        cc->dump_state(cpu, f, flags);   //  <===== qemu_fprintf
    }
}
```

`cpu_dump_state`首先进行一些同步相关的函数`cpu_synchronize_state()`，之后执行主功能函数`cc->dump_state`。

```c
void cpu_synchronize_state(CPUState *cpu)
{
    if (cpus_accel->synchronize_state) {
        cpus_accel->synchronize_state(cpu);
    }
}

void x86_cpu_dump_state(CPUState *cs, FILE *f, int flags)
{
	...
	qemu_fprintf(f, "RAX=%016" PRIx64 " RBX=%016" PRIx64 " RCX=%016" PRIx64 " RDX=%016" PRIx64 "\n"
				 "RSI=%016" PRIx64 " RDI=%016" PRIx64 " RBP=%016" PRIx64 " RSP=%016" PRIx64 "\n"
				 "R8 =%016" PRIx64 " R9 =%016" PRIx64 " R10=%016" PRIx64 " R11=%016" PRIx64 "\n"
				 "R12=%016" PRIx64 " R13=%016" PRIx64 " R14=%016" PRIx64 " R15=%016" PRIx64 "\n"
	...
}
```

可见不是任何情况都需要做`cpu_synchronize_state`操作的(下面会介绍kvm和tcg下的情况)。而x86虚拟机的`cc->dump_state = x86_cpu_dump_state`，主要是用`qemu_fprintf`这个qapi打印信息


## cpu work item

kvm下可以将事件添加到cpu的work list中，vCPU线程会负责处理work list中的事件。

### kvm

启用kvm的情况下会调用`cpus_accel->synchronize_state`"同步函数"，为`kvm_cpu_synchronize_state`：

```c
void kvm_cpu_synchronize_state(CPUState *cpu)
{
    if (!cpu->vcpu_dirty) {
        run_on_cpu(cpu, do_kvm_cpu_synchronize_state, RUN_ON_CPU_NULL); 
    }
}
```

`run_on_cpu`调用`do_run_on_cpu`让主循环进入等待，等待vCPU线程处理，从而实现同步：

```c
void do_run_on_cpu(CPUState *cpu, run_on_cpu_func func, run_on_cpu_data data,
                   QemuMutex *mutex)
{
    struct qemu_work_item wi;

    if (qemu_cpu_is_self(cpu)) {
        func(cpu, data);
        return;
    }

    wi.func = func;
    wi.data = data;
    wi.done = false;
    wi.free = false;
    wi.exclusive = false;

    queue_work_on_cpu(cpu, &wi);
    while (!qatomic_mb_read(&wi.done)) {
        CPUState *self_cpu = current_cpu;

        qemu_cond_wait(&qemu_work_cond, mutex);
        current_cpu = self_cpu;
    }
}
```

`queue_work_on_cpu()`将一个work item加入cpu的work list尾部，下面的while循环中最终会调用pthread库的`pthread_cond_wait()`函数等待vCPU调用`qemu_wait_io_event()`处理，从而实现主循环线程和vCPU线程的同步。

```c
static void *kvm_vcpu_thread_fn(void *arg)
{
	...
    do {
        if (cpu_can_run(cpu)) {
            r = kvm_cpu_exec(cpu); 				// vCPU主函数
            if (r == EXCP_DEBUG) {
                cpu_handle_guest_debug(cpu);
            }
        }
        qemu_wait_io_event(cpu); 				// 处理work list
    } while (!cpu->unplug || cpu_can_run(cpu));
	...
}
```


## 全局锁

全局锁是tcg和kvm下都存在的同步方式，下面用tcg下输入`info register`命令说明同步机制

### tcg

tcg无法通过`cpus_accel->synchronize_state`判断不会像kvm一样有特殊的同步方法。通过gdb调试，**猜测tcg的同步通过全局锁完成**，证明如下：

```c
if (cpus_accel->synchronize_state) {
	cpus_accel->synchronize_state(cpu);
}
```

如下命令启动虚拟机，在`cpu_dump_state`初打上断点，`info register`触发。

```sh
gdb	 \
  --ex 'b cpu_dump_state'\
  --args \
qemu-system-x86_64 \
  -M pc \
  -m 8G \
  -kernel ./vmlinux \
```

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_monitor_model/dispatch_monitor.png" alt="">

切换至vCPU线程`thread 3`，发现其似乎在等待锁。当前位置打上断点，观察何时会触发：

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_monitor_model/vcpu_wait.png" alt="">

回到主循环，一路finish执行，观察vCPU线程断点的触发情况。发现在`qemu_mutex_unlock(&qemu_global_mutex);`后断点触发，即主循环释放锁后vCPU线程继续执行。

<img src="https://raw.githubusercontent.com/66RING/66RING/master/.github/images/os_proj/qemu_monitor_model/unlock_hit.png" alt="">

解释：注意到main loop中是先unlock在lock的，而vCPU线程主函数也有全局锁保护

```c
qemu_main_loop -> main_loop_wait -> os_host_main_loop_wait

static int os_host_main_loop_wait(int64_t timeout)
{
    ...
    qemu_mutex_unlock_iothread();
    ret = qemu_poll_ns((GPollFD *)gpollfds->data, gpollfds->len, timeout);
	...
    qemu_mutex_lock_iothread();
	glib_pollfds_poll(); 		//  --> dispatch()
	...
}

static void *tcg_cpu_thread_fn(void *arg)
{
	...
	qemu_mutex_unlock_iothread();
	r = tcg_cpu_exec(cpu);
	qemu_mutex_lock_iothread();
	...
	qemu_wait_io_event(cpu);
	...
}
```

主循环和vCPU都是在while循环中，锁保护的是主循环的`dispatch`和vCPU线程的`qemu_wait_io_event`。而主循环的poll和vCPU线程的`cpu_exec`和同时发生。

因此monitor作为一个iothread，`mon_iothread`，在主循环dispatch中进行，vCPU想要`cpu_exec`之后的操作需要等待下一次主循环poll发生。
