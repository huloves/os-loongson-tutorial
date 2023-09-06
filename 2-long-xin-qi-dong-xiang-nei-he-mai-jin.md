# 2 龙芯启动，向内核迈进

### 龙芯启动部分程序，向内核迈进

从计算机的启动到执行C语言程序大致流程为：1.计算机上电 `->` 2.运行 ROM 程序（BIOS、UEFI） `->` 3.从 ROM 程序接管计算机控制，进行初步设置 `->` 4.运行 C 语言程序。这其中内核编程人员从第 3 步开始需要编写程序，且这部分代码基本上为直接操作寄存器进行设置，所以一般使用汇编代码编写。当对计算机进行内核所需的初始化之后，跳转到 C 语言程序部分开始执行后续内容。后续内容中一般只有少部分代码需要直接操作寄存器，所以后续直接操作寄存器的内容一般使用内敛汇编完成。

> 如果后续内容中存在需要大量操作寄存器的部分，也会直接编写汇编程序。例如例外和中断处理、上下文切换。

本章龙芯启动部分程序的内容为：从ROM程序开始接管计算机控制，进行初步设置，然后跳转至C语言程序。C语言程序中使用串口输出功能，输出`hello os-loongson`。

### 2.1 龙芯启动

上面提到的计算机启动大致流程中，第 3 步的一个作用是为内核提供一个确定性的状态，方便内核管理计算机。为了给内核提供确定性的状态，所以要设置很多系统寄存器和通用寄存器，就会用到汇编语言。为了先体验一下在龙芯架构上的开发，使用最简单的方法看到执行效果。这里先简化这个过程，不进行这个第 3 步的初始化。计算机上电、运行 ROM 程序之后，写一个简单的串口输出程序的 C 语言程序，直接输出 `hello loongson` 字符串。

> 可参考 [https://github.com/huloves/os-loongson](https://github.com/huloves/os-loongson) 的 ch2-1 分支

修改 os-loongson/os-elephant-dev/kernel/init.c 文件内容：

```c
/* os-loongson/os-elephant-dev/kernel/init.c */
// #include "init.h"
// #include "print.h"
// #include "interrupt.h"
// #include "timer.h"
// #include "memory.h"
// #include "thread.h"
// #include "console.h"
// #include "keyboard.h"
// #include "tss.h"
// #include "syscall-init.h"
// #include "ide.h"
// #include "fs.h"

unsigned long uart_base = 0x1fe001e0;   // UART0 寄存器物理地址基址

#define UART0_THR  (uart_base + 0)   // 数据传输寄存器
#define UART0_LSR  (uart_base + 5)   // 线路状态寄存器
#define LSR_TX_IDLE  (1 << 5)	     // 传输 FIFO 位空表示位，为 1 表示可以传输

static char io_readb(unsigned long addr)
{
	return *(volatile char*)addr;
}

static void io_writeb(unsigned long addr, char c)
{
	*(char*)addr = c;
}

static void putc(char c)
{
	// wait for Transmit Holding Empty to be set in LSR.
	while((io_readb(UART0_LSR) & LSR_TX_IDLE) == 0);
	io_writeb(UART0_THR, c);
}

static void puts(char *str)
{
	while (*str != 0) {
		putc(*str);
		str++;
	}
}

/* 负责初始化所有模块 */
void init_all()
{
	puts("hello os-loongson\n");
	while(1);
	// put_str("init_all\n");
	// idt_init();	     // 初始化中断
	// mem_init();	     // 初始化内存管理系统
	// thread_init();    // 初始化线程相关结构
	// timer_init();     // 初始化PIT
	// console_init();   // 控制台初始化最好放在开中断之前
	// keyboard_init();  // 键盘初始化
	// tss_init();       // tss初始化
	// syscall_init();   // 初始化系统调用
	// intr_enable();    // 后面的ide_init需要打开中断
	// ide_init();	     // 初始化硬盘
	// filesys_init();   // 初始化文件系统
}
```

暂时注释掉原有的代码，新增的代码就是串口输出的一个简单实现。上面代码中涉及的知识点和与 x86 架构 os-elephant-dev 的不同点：

* 外设IO端口编址方式：统一编址、独立编址。
* 串口以及读写串口的方法。
* x86 架构 os-elephant-dev 在启动后输出字符为访问 0xb8000 起始的显存。

创建 os-loongson/os-elephant-dev/script 目录，然后在该目录下创建一个名为 kernel.ld 的文件。

```bash
os-loongson/os-elephant-dev$ mkdir script
os-loongson/os-elephant-dev$ cd script
os-loongson/os-elephant-dev/script$ touch kernel.ld
```

kernel.ld 的内容涉及链接脚本的知识，推荐博客及官方文档：

* 博客：[https://www.cnblogs.com/jianhua1992/p/16852784.html](https://www.cnblogs.com/jianhua1992/p/16852784.html)
* 官方文档：[https://sourceware.org/binutils/docs/ld/index.html#SEC\_Contents](https://sourceware.org/binutils/docs/ld/index.html#SEC\_Contents)

kernel.ld 文件的内容如下：

```c
ENTRY(init_all)   /* 设置入口点 */
SECTIONS
{
        . = 0x0000000000200000;
	PROVIDE( _start = . );
        .head.text : {
                KEEP(*(.head.text))
        }
	.text : {
		*(.text)
		. = ALIGN(4096);
	}
	. = ALIGN(1 << 12);
	.data : {
                __start_init_task = .;
                init_thread_union = .; 
                init_stack = .;
                . = __start_init_task + 0x00004000;
		*(.data)
		*(.rodata)
		. = ALIGN(8192);
	}
	. = ALIGN(1 << 12);
        __bss_start = .;
	.bss : {
		*(.bss)
		. = ALIGN(4096);
	}
        __bss_stop = .;
	. = ALIGN(1 << 12);
	PROVIDE( _end = . );
}
```

> 为什么这里把起始地址安排在 0x0000000000200000 ？
>
> 参阅文档 loongson\_devsys\_firmware\_kernel\_interface\_specification\_v2.2.pdf 第 4 节。
>
> 该文档下载地址：[https://github.com/loongson-community/docs/blob/master/firmware/loongson\_devsys\_firmware\_kernel\_interface\_specification\_v2.2.pdf](https://github.com/loongson-community/docs/blob/master/firmware/loongson\_devsys\_firmware\_kernel\_interface\_specification\_v2.2.pdf)

修改 os-loongson/os-elephant-dev/makefile 文件，修改为如下内容：

```makefile
BUILD_DIR = ./build
CROSS_COMPILE = ../toolchains/cross-tools/bin/loongarch64-unknown-linux-gnu-
AS = $(CROSS_COMPILE)as
CC = $(CROSS_COMPILE)gcc
LD = $(CROSS_COMPILE)ld
CFLAGS = -Wall -g -march=loongarch64 -mabi=lp64s -ffreestanding -fno-common -nostdlib -fno-stack-protector -fno-pie -no-pie -c -I ./include/
AFLAGS = -nostdinc -D__ASSEMBLY__ -fno-PIE -mabi=lp64s -G0 -pipe -msoft-float -mexplicit-relocs -ffreestanding -mno-check-zero-division -c -I ./include/
LDFLAGS = -m elf64loongarch --no-warn-rwx-segments -G0 -n -nostdlib 

OBJS = $(BUILD_DIR)/init.o

$(BUILD_DIR)/init.o: kernel/init.c
	$(CC) $(CFLAGS) $< -o $@

$(BUILD_DIR)/kernel.elf: $(OBJS)
	$(LD) $(LDFLAGS) -T ./script/kernel.ld $^ -o $@

clean:
	cd $(BUILD_DIR) && rm -rf ./* && if [ -e hd*.img  ];then rm ../*.img;fi

build_dir:
	mkdir -p $(BUILD_DIR)

build: $(BUILD_DIR)/kernel.elf

all: build_dir clean build
```

在 os-loongson/os-elephant-dev/ 目录下执行 make all 进行编译链接：

```bash
os-loongson/os-elephant-dev$ make all
```

在 os-loongson/qemu/ 目录下执行下述命令运行：

```bash
os-loongson/qemu$ ./qemu-system-loongarch64 -m 4G -smp 1 -bios ./loongarch_bios_0310_debug.bin -vga none -nographic -kernel ../os-elephant-dev/build/kernel.elf
```

运行截图如下所示：

<figure><img src=".gitbook/assets/WX20230828-234605.png" alt="" width="346"><figcaption><p>图 2.1 串口输出运行截图</p></figcaption></figure>

### 2.2 内核初始化，提供确定性状态

内核在刚开始运行时需要一些确定性的设置，包括运行在哪个特权等级下、内存地址访问方式。这里会涉及龙芯架构的内容：

* 特权等级
* MMU 支持的两种虚实地址翻译模式
* 访问存储类型
* 控制状态寄存器（CSR）

先介绍一下龙芯架构的内容，然后开始写代码进行实践。

#### 2.2.1 特权等级

龙芯架构中处理器核分为 4 个特权等级（Privilege LeVel，简称 PLV），分别是 PLV0\~PLV3。所有特权等级中，PLV0 是具有最高权限的特权等级，也是唯一可以使用特权指令并访问所有特权资源的特权等级。

> 所有特权指令仅在 PLV0 特权等级下才能访问。仅有一个例外情况，当 CSR.MISC 中的 RPCNTL1、RPCNTL2、RPCNTL3 配置为 1 时，可以在 PLV1、PLV2、PLV3 特权等级下执行 CSRRD 指令读取性 能计数器。

#### 2.2.2 虚实地址翻译模式

虚实地址翻译模式分为**直接地址翻译模式**和**映射地址翻译模式**。映射地址翻译模式分为**直接映射地址翻译模式**（简称“直接映射模式”）和**页表映射地址翻译模式**（页表映射模式）。

直接映射配置窗口（DMW0\~DMW3），这一组寄存器参与完成直接映射地址翻译模式。

<table data-full-width="false"><thead><tr><th width="103" align="center">位</th><th width="106" align="center">名字</th><th width="93" align="center">读写</th><th align="center">描述</th></tr></thead><tbody><tr><td align="center">0</td><td align="center">PLV0</td><td align="center">RW</td><td align="center">为 1 表示在特权等级 PLV0 下可以使用该窗口的配置进行直接映射地址翻译。</td></tr><tr><td align="center">1</td><td align="center">PLV1</td><td align="center">RW</td><td align="center">为 1 表示在特权等级 PLV1 下可以使用该窗口的配置进行直接映射地址翻译。</td></tr><tr><td align="center">2</td><td align="center">PLV2</td><td align="center">RW</td><td align="center">为 1 表示在特权等级 PLV2 下可以使用该窗口的配置进行直接映射地址翻译。</td></tr><tr><td align="center">3</td><td align="center">PLV3</td><td align="center">RW</td><td align="center">为 1 表示在特权等级 PLV3 下可以使用该窗口的配置进行直接映射地址翻译。</td></tr><tr><td align="center">5:4</td><td align="center">MAT</td><td align="center">RW</td><td align="center">虚地址落在该映射窗口下访存操作的存储访问类型。</td></tr><tr><td align="center">59:6</td><td align="center">0</td><td align="center">R0</td><td align="center">保留域。读返回 0，且软件不允许改变其值。</td></tr><tr><td align="center">63:60</td><td align="center">VSEG</td><td align="center">RW</td><td align="center">直接映射窗口虚地址的 [63:60] 位。</td></tr></tbody></table>

#### 2.2.3 存储访问类型

龙芯架构下支持三种存储访问类型，分别是：一致可缓存（Coherent Cached，简 称 CC）、强序非缓存（Strongly-ordered UnCached，简称 SUC）和弱序非缓存（Weakly-ordered UnCached， 简称 WUC）。

当处理器核 MMU 处于映射地址翻译模式时，存储访问类型的确定分为两种情况。如果取指或 load/store 操作的地址落在某个**直接映射配置窗口**上，那么该取指或 load/store 操作的存储访问类型由配置该窗口的 CSR 寄存器中的 MAT 域决定。如果取指或 load/store 只能**通过页表完成映射**，那么其存储访问类型由页表 项中的 MAT 域决定。MAT 域的值和存储访问类型之间的关系如下表所示：

| MAT 值 | 存储访问类型 |
| :---: | :----: |
|   0   |  强序非缓存 |
|   1   |  一致可缓存 |
|   2   |  弱序非缓存 |
|   3   |   保留   |

> 龙芯架构参考手册卷一，2.1.7节到2.1.9节详细介绍龙芯架构的存储访问类型。

#### 2.2.4 控制状态寄存器（CSR）

特权资源中非常重要的是控制状态寄存器，控制状态寄存器控制着系统的状态。控制状态寄存器拥有独立的空间，在龙芯架构参考手册中的第 7.1 节中有控制状态寄存器的地址及其名称。访问控制状态寄存器使用 csrrd 指令、csrwr 指令和 csrchg 指令。在后面遇到某一个控制状态寄存器时再展开讲述其内容。

#### 2.2.5 LoongArch 汇编部分内容

* 访存指令：st、ld
* 访问控制状态寄存器指令：csrrd、csrwr
* 跳转指令：b
* 运算指令：add
* 加载地址：la
* 加载立即数：li

#### 2.2.6 内核运行环境确定性设置及传递启动参数

开始使用汇编语言了，就会使用到通用寄存器。龙芯架构的通用寄存器有 32 个，记为 r0 \~ r31。LA32 中每个寄存器为 32 位宽， LA64 中每个寄存器为 64 位宽。r0 的内容总是固定为 0，而其他寄存器在体系结构层面没有特殊功能。（ r1 算是一个例外，在 BL 指令中固定用作链接返回寄存器。）

虽然大多通用寄存器在体系结构层面没有特殊功能，但是我们使用的工具链符合 LoongArch ELF psABI 规范，该规范中对通用寄存器的使用进行了约定，约定如下：

<table><thead><tr><th width="177">寄存器名</th><th width="162">别名</th><th width="178">用途</th><th>跨调用保持</th></tr></thead><tbody><tr><td><code>$r0</code></td><td><code>$zero</code></td><td>常量0</td><td>不使用</td></tr><tr><td><code>$r1</code></td><td><code>$ra</code></td><td>返回地址</td><td>否</td></tr><tr><td><code>$r2</code></td><td><code>$tp</code></td><td>TLS/线程信息指针</td><td>不使用</td></tr><tr><td><code>$r3</code></td><td><code>$sp</code></td><td>栈指针</td><td>是</td></tr><tr><td><code>$r4</code>-<code>$r11</code></td><td><code>$a0</code>-<code>$a7</code></td><td>参数寄存器</td><td>否</td></tr><tr><td><code>$r4</code>-<code>$r5</code></td><td><code>$v0</code>-<code>$v1</code></td><td>返回值</td><td>否</td></tr><tr><td><code>$r12</code>-<code>$r20</code></td><td><code>$t0</code>-<code>$t8</code></td><td>临时寄存器</td><td>否</td></tr><tr><td><code>$r21</code></td><td><code>$u0</code></td><td>每CPU变量基地址</td><td>不使用</td></tr><tr><td><code>$r22</code></td><td><code>$fp</code></td><td>帧指针</td><td>是</td></tr><tr><td><code>$r23</code>-<code>$r31</code></td><td><code>$s0</code>-<code>$s8</code></td><td>静态寄存器</td><td>是</td></tr></tbody></table>

龙芯架构的通用寄存器汇编中使用龙芯架构的通用寄存器时，使用 `$r0` \~ `$r31` 这样的命名方式。为了方便通用寄存器的理解和使用，先用C语言的宏定义将寄存器名和别名关联起来：

```c
/* os-elephant-dev/include/regdef.h */
#ifndef _REGDEF_H
#define _REGDEF_H

#define zero	$r0	/* 常量0 */
#define ra	$r1	/* 返回地址 */
#define tp	$r2
#define sp	$r3	/* 栈指针 */
#define a0	$r4	/* 参数寄存器, a0/a1重用为v0/v1, 作为返回值 */
#define a1	$r5
#define a2	$r6
#define a3	$r7
#define a4	$r8
#define a5	$r9
#define a6	$r10
#define a7	$r11
#define t0	$r12	/* 临时寄存器, 调用者保存, 函数内直接使用 */
#define t1	$r13
#define t2	$r14
#define t3	$r15
#define t4	$r16
#define t5	$r17
#define t6	$r18
#define t7	$r19
#define t8	$r20
#define u0	$r21
#define fp	$r22	/* 帧指针 */
#define s0	$r23	/* 静态寄存器, 子程序使用需要保存和恢复 */
#define s1	$r24
#define s2	$r25
#define s3	$r26
#define s4	$r27
#define s5	$r28
#define s6	$r29
#define s7	$r30
#define s8	$r31

#endif /* _ASM_REGDEF_H */
```

所有的通用寄存器这样就使用 C 语言宏，将寄存器名和别名做了关联。接下来将马上要使用到的控制状态寄存器也用 C 语言宏表示出来。

```c
#ifndef _LOONGARCH_H
#define _LOONGARCH_H

#define LONGSIZE	8

#define DMW_PABITS	48

#ifdef __ASSEMBLY__
#define _CONST64_(x)	x
#else
#define _CONST64_(x)	x ## L
#endif

/* Basic CSR registers */
#define LOONGARCH_CSR_CRMD		0x0	/* 当前模式信息 */
#define LOONGARCH_CSR_PRMD		0x1	/* 例外前模式信息 */
#define LOONGARCH_CSR_EUEN		0x2	/* 扩展部件使能 */

/* KSave registers */
#define LOONGARCH_CSR_KS0		0x30
#define LOONGARCH_CSR_KS1		0x31
#define LOONGARCH_CSR_KS2		0x32
#define LOONGARCH_CSR_KS3		0x33
#define LOONGARCH_CSR_KS4		0x34
#define LOONGARCH_CSR_KS5		0x35
#define LOONGARCH_CSR_KS6		0x36
#define LOONGARCH_CSR_KS7		0x37
#define LOONGARCH_CSR_KS8		0x38

/* Percpu-data base allocated KS3 statically */
#define PERCPU_BASE_KS			LOONGARCH_CSR_KS3

/* Direct Map windows registers */
#define LOONGARCH_CSR_DMWIN0		0x180	/* 直接映射配置窗口0 */
#define LOONGARCH_CSR_DMWIN1		0x181	/* 直接映射配置窗口1 */

/* 直接映射配置窗口0/1的配置 */
#define CSR_DMW0_PLV0		_CONST64_(1 << 0)
#define CSR_DMW0_VSEG		_CONST64_(0x8000)
#define CSR_DMW0_BASE		(CSR_DMW0_VSEG << DMW_PABITS)
#define CSR_DMW0_INIT		(CSR_DMW0_BASE | CSR_DMW0_PLV0)

#define CSR_DMW1_PLV0		_CONST64_(1 << 0)
#define CSR_DMW1_MAT		_CONST64_(1 << 4)
#define CSR_DMW1_VSEG		_CONST64_(0x9000)
#define CSR_DMW1_BASE		(CSR_DMW1_VSEG << DMW_PABITS)
#define CSR_DMW1_INIT		(CSR_DMW1_BASE | CSR_DMW1_MAT | CSR_DMW1_PLV0)

#endif /* _LOONGARCH_H */
```

暂时用到的寄存器就已经表示完了，接下来开始编写启动设置寄存器的代码。编写 os-loongson/os-elephant-dev/boot/loongarch/head.S 文件：

{% code fullWidth="false" %}
```c
#include <regdef.h>
#include <loongarch.h>
#include <bootinfo.h>

.section ".head.text","ax"

	.align 12

.global kernel_entry
kernel_entry:
	/* Config direct window and set PG */
	li.d		t0, CSR_DMW0_INIT	# UC, PLV0, 0x8000 xxxx xxxx xxxx
	csrwr		t0, LOONGARCH_CSR_DMWIN0
	li.d		t0, CSR_DMW1_INIT	# CA, PLV0, 0x9000 xxxx xxxx xxxx
	csrwr		t0, LOONGARCH_CSR_DMWIN1

	/* We might not get launched at the address the kernel is linked to,
	   so we jump there.  */
	la.abs		t0, 0f
	jr		t0
0:
	/* Enable PG */
	li.w		t0, 0xb0		# PLV=0, IE=0, PG=1
	csrwr		t0, LOONGARCH_CSR_CRMD
	li.w		t0, 0x04		# PLV=0, PIE=1, PWE=0
	csrwr		t0, LOONGARCH_CSR_PRMD
	li.w		t0, 0x00		# FPE=0, SXE=0, ASXE=0, BTE=0
	csrwr		t0, LOONGARCH_CSR_EUEN

	la.pcrel	t0, __bss_start		# clear .bss
	st.d		zero, t0, 0
	la.pcrel	t1, __bss_stop
1:
	addi.d		t0, t0, LONGSIZE
	st.d		zero, t0, 0
	bne		t0, t1, 1b

	la.pcrel	t0, fw_arg0
	st.d		a0, t0, 0		# firmware arguments
	la.pcrel	t0, fw_arg1
	st.d		a1, t0, 0
	la.pcrel	t0, fw_arg2
	st.d		a2, t0, 0

	/* KSave3 used for percpu base, initialized as 0 */
	csrwr		zero, PERCPU_BASE_KS
	/* GPR21 used for percpu base (runtime), initialized as 0 */
	move		u0, zero

	la.pcrel	tp, init_thread_union
	/* Set the SP after an empty pt_regs.  */
	li.d		sp, (KERNEL_STACK_SIZE - PT_SIZE)
	add.d		sp, sp, tp
	/* set_saved_sp	sp, t0, t1 */
	la.abs		t0, kernelsp
	st.d		sp, t0, 0

	bl		init_all
```
{% endcode %}

上面用到的一些变量定义在 setup.c 或者链接脚本中，使用 bootinfo.h 文件向外部提供。

bootinfo.h 文件的内容如下：

<pre class="language-c"><code class="lang-c"><strong>/* os-loongson/os-elephant-dev/include/bootinfo.h */
</strong><strong>#ifndef _BOOTINFO_H
</strong>#define _BOOTINFO_H

#define KERNEL_STACK_SIZE	0x00004000   // 16K
#define PT_SIZE			328

#ifndef __ASSEMBLY__

extern unsigned long fw_arg0, fw_arg1, fw_arg2;
extern unsigned long kernelsp;

union thread_union {
	unsigned long stack[KERNEL_STACK_SIZE / sizeof(unsigned long)];
};

extern union thread_union init_thread_union;

extern unsigned long init_stack[KERNEL_STACK_SIZE / sizeof(unsigned long)];

#endif /* !__ASSEMBLY__ */

#endif /* _BOOTINFO_H */
</code></pre>

setup.c 文件的内容如下：

```c
/* os-loongson/os-elephant-dev/boot/loongarch/setup.c */
#include <bootinfo.h>

unsigned long fw_arg0, fw_arg1, fw_arg2;
unsigned long kernelsp;
```

修改 kernel.ld 文件的内容如下：

```c
/* os-loongson/os-elephant-dev/script/kernel.ld */
ENTRY(init_all)   /* 设置入口点 */
SECTIONS
{
        . = 0x0000000000200000;
	PROVIDE( _start = . );
        .head.text : {
                KEEP(*(.head.text))
        }
	.text : {
		*(.text)
		. = ALIGN(4096);
	}
	. = ALIGN(1 << 12);
	.data : {
                __start_init_task = .;
                init_thread_union = .; 
                init_stack = .;
                . = __start_init_task + 0x00004000;
		*(.data)
		*(.rodata)
		. = ALIGN(8192);
	}
	. = ALIGN(1 << 12);
        __bss_start = .;
	.bss : {
		*(.bss)
		. = ALIGN(4096);
	}
        __bss_stop = .;
	. = ALIGN(1 << 12);
	PROVIDE( _end = . );
}
```

修改 makefile 文件：

```makefile
OBJS = $(BUILD_DIR)/head.o $(BUILD_DIR)/init.o $(BUILD_DIR)/setup.o

$(BUILD_DIR)/head.o: boot/loongarch/head.S
	$(CC) $(AFLAGS) $< -o $@

$(BUILD_DIR)/setup.o: boot/loongarch/setup.c
	$(CC) $(CFLAGS) $< -o $@

$(BUILD_DIR)/init.o: kernel/init.c
	$(CC) $(CFLAGS) $< -o $@
```

在 os-loongson/os-elephant\_dev/ 目录下编译，然后到 os-loongson/qemu/ 目录下运行：

```bash
os-loongson/os-elephant_dev$ make all
os-loongson/os-elephant_dev$ cd ../qemu/
os-loongson/qemu$ ./qemu-system-loongarch64 -m 4G -smp 1 -bios ./loongarch_bios_0310_debug.bin -vga none -nographic -kernel ../os-elephant-dev/build/kernel.elf
```
