---
title: 嵌入式系统课程笔记
author: 66RING
date: 2000-01-01
mathjax: true
---


# 自主可控嵌入式系统课程提纲

## 第一课

1. 嵌入式系统的定义；
	- 应应用为中心、计算机技术为基础，软硬件可裁剪，适应应用系统对功能、可靠性、成本、体积、功耗等严格要求的专用计算机系统
2. 嵌入式系统和通用计算机的区别；
	a. 专用性: 专门的处理器、功能算法的专用性、系统对用户透明
	b. 小型化(资源有限): 结构紧凑、资源有限
	c. 软硬件设计一体化: 软硬间依赖强，一般协同设计
	d. 需要交叉开发环境: 开发需要在宿主机完成
3. 嵌入式实时系统的特点；
	a. 专用性: 专门的处理器、功能算法的专用性、系统对用户透明
	b. 小型化(资源有限): 结构紧凑、资源有限
	c. 软硬件设计一体化: 软硬间依赖强，一般协同设计
	d. 需要交叉开发环境: 开发需要在宿主机完成
4. 嵌入式系统的应用领域；
	a. 工业控制
	b. 信息化时代
	c. 物联网
	d. 定制芯片、算法优化
5. 嵌入式系统的三要素；
	a. 嵌入性: 嵌入到对象体系中，有对象环境要求
	b. 专用性: 软硬件按对象需求裁剪
	c. 计算机: 实现对象的智能化功能
6. 嵌入式处理器的分类。
	a. 表现形式分类
		i. 系统级
		ii. 板级
		iii. 片级
	b. 实时性要求分类
		i. 非实时， 如PDF
		ii. 软实时， 超时性能下降，消费类产品
		iii. 硬实时，超时导致失败，工业和军工

## 第二课

1. 冯诺依曼和哈弗体系结构的区别
	a. 双体
	b. 双独立总线
2. 组合逻辑型和存储逻辑型控制器的区别
	a. 组合逻辑型(RISC)
		i. 指令类型少、功能简单、寻址方式少、硬布线
		ii. 规整
	b. 存储逻辑型(CISC)
		i. 微指令
		ii. 复杂
3. 理解流水线、CPU字长对CPU性能的影响
	a. 指令并行执行
	b. cpu效率
	c. 多流水线
	d. **CPU字长: 微处理器一次能处理的数据宽度**
4. CPU寄存器与内存的区别
	a. 内置于cpu
	b. 速度区别
5. RISC和CISC的主要区别（指令集、流水线、寄存器、Load/Store结构四个方面）
	a. CISC复杂指令集计算机，RISC精简指令集计算机
| 指标       | RISC                             | CISC                        |
|------------|----------------------------------|-----------------------------|
| 指令集     | 一周期一指令;简单组复杂;固定长度 | 长度不固定;执行需要多个周期 |
| 流水线     | 每周期前进一步;易流水线          | 指令执行需要调用微程序      |
| 寄存器     | 多通用寄存器                     | 特定目的的专用寄存器        |
| load/store | 独立的l/s指令                    | 处理器直接处理存储器的数据  |

## 第三课

1. ARM的历史（了解）
	a. arm = advanced risc machine
	b. arm = 公司名 = 处理器架构 = 技术 = 商标
	c. 特点
		- 小体积、低功耗
		- L/S
		- 16/32双指令集
		- 3地址指令格式
	d. 字32bit
2. Thumb工作状态（了解）
	a. 16bit指令长度
	b. 32bit执行效率
3. ARM的7种运行模式（用户、系统、5种异常模式）
	a. usr、sys、fiq、irq、svc(管理)、abt(中止)、und(未定义)
	b. 除了用户外都是特权
	c. 除用户和系统外都是异常
4. 37个寄存器的组织形式（R13、R14、R15以及状态寄存器的作用）
	a. 1 pc
	b. 30 通用
	c. 1 cpsr
	d. 5 spsr
	e. R0~R7未分组
	f. R8~R12专门多给fir一组
	g. R13, R14, SPSR异常模式各有一组，快速返回
		- (非异常模式没有SPSR)
	h. R9(SB), R10(SL), R11(FP)
	i. **R12(IP), R13(SP), R14(LR)**
	j. R15(PC)


## 第四课

基本指令格式

```
<opcode>{cond}{S}<Rd>, <Rn>{,<operand2>}
```
- 前4bit(31~28)都是用于表示条件的
- {S}表示是否影响标志位，有S将影响
- `!`的作用是写回使能, 数据传送完毕后将最后的地址写入基地址寄存器
- e.g. `ADDNE`会判断标志符再执行

1. 充分理解Load/Store结构
2. 指令分类（主要掌握数据处理类、Load/Store类和跳转类）
	- 数据处理指令
	- load/store指令
	- 跳转指令
	- CPSR处理指令
	- 异常产生指令
	- 协处理指令
3. 掌握3地址指令格式，理解条件码和条件执行；
	- 条件码: 高4bit
	- 三地址指令格式: 4(dst), 4(src), 12(imm) => **第二个源操作数细节**
4. 掌握寻址方式，重点理解立即寻址、寄存器寻址、寄存器移位寻址、基指寻址和变址寻址。
	- 立即寻址: **只有第二源操作数可以立即数**, `#`前缀
		* **立即数**: 8(常数) + 4(**偶数移位个数**:刚好表示32bit), 0x0001_0000_0010不合法, 0x0010_0000_0100合法
	- 寄存器寻址：`mov r0, r1, r2`: r0 = r1 + r2
	- **寄存器移位寻址**: `add r3, r2, r1, LSL #3`: `r3=r2+(r1>>3)`
		* 因为12bit有冗余, 第三个寄存器只用4bit还是8bit = 3(type) + 5(移位数值)
	- 基址寻址: 针对load/store: `ldr/str r0, [r1]`
		* 注意数据传送方向：`ldr r0, {r1-r4}`将r0指示的4个存储器传送到r1-r4. `str r0, {r1-r4}`将r1-r4保存到r0指示的存储器中
	- 变址寻址: `ldr/str r0, [r1, r2, #4]`
		* **U控制位**, 1时加偏移，0时减偏移


## 第五课

1. 了解具体指令。

- 变址模式
- 后缀`!`
- 块拷贝寻址STM, LDM
	* IA, IB, DA, DB  &  FD, FA, ED, EA.
		+ Increase after/before都是相对传输指令的after/before
	* 压栈，出栈的操作数顺序(区分递减栈，递增栈)

- mov
- mvn
- add, adc
- sub, sbc
- rsb, rsc 逆向减
- and, **orr**, eor, bic
- cmp, cmn, tst, teq
- mul, mla
- **L/S**
	* ldr, ldrb(B), ldrh(2B)
	* str, strb, strh
	* ldm, sdm
	* swp, swpb
		+ swp r0, r1, r2 => r0 <- r2, r2 <- r1
- **B指令**
	* b, bl(r14), **bx(thumb)**, bxl
- 状态寄存器
	* mrs(r<-s), msr(s<-r)
- 异常产生指令
	* swi
	* bkpt
- 协处理器指令
	* cdp, ldc, stc, mcr, mrc


## 第六课

1. 了解ARM汇编，看课件最后的几个汇编示例。

### 符号定义

- gbla, gbll, gbls
- lcla, lcll, lcls
- 寄存器变量：`名 RLST {r0-r5, r8, r10}`

### 数据定义

- storg, ltorg
- map/`^`
	* `map 0x100, r0`
- field/`#`
	```
	定义数据结构
	map 0x100
	A filed 16
	B filed 32
	C filed 256
	```
- space/`%`
	* `DataSpace % 100`(初值0, B)
- dcb(B), dcw(半字), dcd/`&`(字), dcfs(float), dcfd(double), dcq(8B)
	* `str DCD "string"`, 指定初始化内容
	* TODO 字半字的意思


### 伪操作

```
if expr
else
endif
```

```
while exp
wend
```

```
macro
	{标号} 宏名 {参数} {...}
mend

mexit   跳出宏
```

###  信息报告

- assert
- info
- opt
- ttl, subt


### 其他

- align
- area
- code
- entry
	* 一个文件只能有一个入口，多个文件导致多入口时，入口由连接器决定
- equ/`*`
	* `test * 0x55, code3`
- export/global
- import/extern
- get/include

```
area 段名, code,readonly,alien=3
	...
end
```

### ARM伪指令

- adr
- adrl
- **LDR**: 将任意一个32bit常数或地址加载到寄存器(即超出imm表示范围时，自动通过缓冲池间接加载)
	* 注意与ldr汇编指令的区别，**注意使用`=`**
	* `LDR R1, =0xff 汇编后：LDR R1, =0xff`
	* `LDR R1, =0xfff 汇编后：LDR R1, [pc, OFFSET_TO_LPOOL]`
- BL，子程序调用(存LR)
- `CMPNE R1, #1`, `ADDEQ R2, R3, R4`


### **程序实例**

```
		AREA Init ，CODE，READONLY ；定义一个代码段
		ENTRY ；标识程序的入口点
start 	LDR R0 ，=0x3FF5000 ；接下来为指令序列
		MOV R1 ，0xFF
		STR R1 ，［R0］
		LDR R0 ，=0x3FF5008
		MOV R1 ，0x01
		STR R1 ，［R0］
			；可以有其他段
			；数据缓冲池的位置
		END ；程序结束
```

子程序调用

```
		AREA Init，CODE，READONLY
		ENTRY
start 	MOV R0 ，#10
		MOV R1 ，#30
		BL subname ；调用子程序subname
return	……
subname	ADD R0，R0，R1 ；子程序
		MOV PC， LR ；从子程序返回
		END
```

条件执行

```
int gcd (int a , int b)
{
	while (a!=b){
		if (a>b)
			a=a-b;
		else
			b=b-a;
	}
	return a;
}

		AREA gcd, CODE, READONLY ALIGN=3
		ENTRY
Start 	CMP R0, R1
		BEQ stop
		BLT less
		SUB R0, R0, R1
		B start
less 	SUB R1, R1, R0
		B start
stop 	NOP
		MOV PC, LR
```

```
		AREA gcd, CODE, READONLY ALIGN=3
		ENTRY
start 	CMP R0, R1 ；比较a和b大小
		SUBGT R0, R0, R1 ；if(a>b) a=a-b
		SUBLT R1, R1, R0；if(a<b) b=b-a
		BNE start ；if(a!=b) 跳转，继续
		MOV PC, LR
```

**信号量操作**

```
	SEM EQU 0x40003000
	…
SEM_WAIT
	MOV R1, #0
	LDR R0, = SEM
	SWP R1, R1, [R0] ;取出信号量， 并设置其为0
	CMP R1, #0 ;判断是否有信号
	BEQ SEM_WAIT ;若没有，则等待
```

**数据块复制**

ppt

**跳转表**

ppt


## 第七课

TODO

- armcc c编译器
- armcpp c++编译器
- armcc 遵循ansi c标准，输出elf格式目标文件

1. 了解C运行时库
	- 运行时库：一种被编译器用来实现编程语言内置函数，以提供该语言程序运行时支持的一种特殊的计算机程序库
	- 如memcpy、printf、malloc
	- arm的c编译器支持ansi c运行时库
	- arm c运行时库是二进制形式提供的
	- 用户可以指定运行时库
	- semihost 利用主机资源实现目标机器上运行的程序所需的输入/输出功能的一些技术
2. 宏和函数的区别
	- 宏名出现的每个地方都编译一次
	- 函数的代码只编译一次
	- 宏不用恢复上下文，也不用返回, 函数相反
	- 简单代码用宏，复杂代码用函数
3. volatile关键字的作用
	- 告诉编译器该变量可能在程序之外修改(如读写端口)
		```
		{
			*port = 0x1			{
			*port = 0x2 	=> 		*port = 0x3
			*port = 0x3			}
		}
		```
	- 不对violatile变量进行优化
	- 不对violatile变量使用缓冲技术
4. 什么是内嵌汇编
	- `__asm`
5. 汇编和C的混合编程（主要理解参数如何传递）
	- 参数可变子程序
		* 参数<=4时，R0~R3传递
		* 参数>4时，可以使用堆栈传递, **eg ppt**
	- 参数固定子程序
		* 第一个整数参数，使用R0~R3传递
		* **其他**出纳书通过堆栈传递
		* 有关浮点运算需要特殊处理
	- 子程序返回
		* 一个32bit整数时，R0
		* 一个64bit整数时，R0和R1，以此类推
6. 了解ARM异常响应过程（理解为主）
	a. 正常执行一条pc+4
	b. 中断发生
		i. 执行完当前指令后，跳转到相应异常中断处理程序
		ii. 执行完成后，返回发生中断时的下一条指令
		iii. 进入异常中断时保留现场，返回时恢复
	c. TODO 中断向量表
	d. TODO arm对异常中断的响应
	e. TODO

## 第八课

1. 了解指令集架构（ISA）和国内ISA生态
	a. TODO
2. RISV-V的主要特征：
	1. 开源
	2. 重新设计（后发优势）
	3. 简单的美学
	4. 模块化的指令集
	5. 指令集可扩展
3. 鲲鹏920使用的指令集架构是**ARM v8 64位**
4. 鲲鹏920生产使用的**制程是7nm**
5. 相比于上一代的鲲鹏916，鲲鹏920处理器的计算核数**提升1倍，最多支持64核**

## 第九课

1. 嵌入式系统最小系统的构成（电源、时钟、复位、存储）
2. SDRAM和Flash（NandFlash、NorFlash）的异同
	- DRAM
		* 电容存储，不断刷新，读写慢，集成高，成本低
	- SDRAM
		* 同cpu和芯片组共享时钟
	- NorFlash
		* 芯片内执行
		* 比NandFlash快
		* 写入、擦除慢，按块进行
	- NandFlash
		* 读写按照512B进行，按块
		* 擦除单元更小



## 第十课（了解为主）

1.外部接口（GPIO控制、触摸屏和LCD）

## 第十一课

1. 嵌入式操作系统的特点
	- **实时性**
		* **确定性**
		* 响应性
		* 响应时间
	- 可移植性
	- 可配置、可裁剪性
	- 可靠性
	- 应用编程接口：每个操作系统系统的系统调用不同
2. 多任务程序设计结构（前后台结构、事件触发结构、操作系统的基本概念）
	- 前台，后台，事件触发
3. 多任务系统的相关概念
	1. 任务的概念（可以和前后台结构做对比）
		* 整个应用被划分为多个任务来处理
	2. 任务的三个基本状态和互相转换关系
		* 运行，阻塞(等待)，就绪
	3. 任务的切换（任务上下文）
		* cpu寄存器的全部内容
	4. 任务的调度（任务级调度、中断级调度）
		* 调度点：任务因资源进入等待，ISR结束
	5. 调度策略（抢占式调度、非抢占式调度）
		* 非抢占: 主动让出，封闭，响应慢
		* 抢占: 响应快，不封闭，需可重入
4. 对临界区的理解（关中断、开中断）
	- 方法有：关中断，禁切换，信号量，CAS
	- 响应时间，开销，注意事项
5. 对共享资源的互斥管理
	- 信号量、消息队列、事件、管道等

## 第十二课

TODO

1.了解uC/OS中任务的状态**（就绪、运行、挂起、休眠和中断）**
	- 就绪
	- 运行
	- 挂起: 等待或延迟
	- 休眠: 任务创建之前的状态，驻留的空间没有还给系统
	- 中断
2.理解任务就绪表的工作机制（优先级位图算法，3种操作）
	- 就绪: 填`OSRdyGrp`和`OSRdyTbl`, bitmap(0~7)置位
	- 挂起: 清零`OSRdyGrp`和`OSRdyTbl`, bitmap(0~7)置位
	- 调度: 根据`OSRdyGrp`和`OSRdyTbl`bitmap，取优先级
3.uC/OS的任务级调度和中断级调度（重在理解） TODO
	- `OSSched()`
	- `OSIntExit()`
4.时钟节拍（重在理解）TODO

- `OS_ENTER_CRITICAL()`, `OS_EXIT_CRITICAL()`
- `<++>`
- `<++>`



## 第十三课（要求能编程实现多任务应用）

1. 任务的创建
	- `OSInit(void)`
	- `OSTaskCreate(task, args ,stack, prio)`
	- `OSStart()`
2. 任务的挂起与恢复机制
	- `OSTaskSuspend(int prio)`
	- `OSTaskResume(int prio)`
3. 任务延迟OSTimeDly的实现机制（利用时钟中断）
	- `OSTimeDly(int ticks)`

## 第十四课（要求能编程应用信号量）

1. uC/OS中事件的概念（事件等待队列的实现机制和任务就绪表一致）
2. uC/OS中信号量的实现机制（创建、P操作和V操作）
	- P: s-- 后 s< 0入等(资源-- 后刚好0, 那就是刚好申请到)
	- V: s++ 后 s<=0出等(资源++ 后仍是0, 那就是最后一个等待刚好出)
	- 信号量cnt是16bit
	- P `OSSemPend`
	- V `OSSemPost`
3. 信号量超时机制。
	- `OSEventTO(OS_EVENT *pevent)`

## 第十五课（理解为主）

1. 优先级反转的概念
	- 优先级继承
	- 优先级天花板
	- ucos -> PIP(优先级继承优先级)融合以上
		* 保留一个优先级做pip
2. uC/OS中互斥型信号量实现机制
	- TODO
3. 了解其它任务间通信机制（邮箱、队列和事件标志组）
	- 邮箱：向另一个任务发送指针变量
	- 消息队列
	- 事件标志组，多对多，一个标志触发多个
4. 了解内存分区管理
	- ucos内存 分区 处理
		* 分配和释放固定大小
		* 大小有几种不同的类型

```
|------------|
| next       |	(void**)addr
| blk dataa  |
|------------|
| next       |
| blk data   |
|------------|
| next       |
| blk data   |
|------------|

用连续内存建立单向链表的过程：

plink = (void**)addr; 				// 获取链表项指针
pblk = (int8*)addr + blksize; 		// 计算下一项地址
for(i = 0; i<(nblk-1); i++) {
	*plink = (void*)pblk; 			// 当前项指向下一项
	plink = (void**)pblk; 			// 更新链表表项指针
	pblk = pblk + blksize; 		 	// 计算下一项地址
}
```

