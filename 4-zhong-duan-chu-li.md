# 4 中断处理

中断处理对现代操作系统有着至关重要的作用，是多任务调度和人机高效交互的基础。操作系统的中断处理需要一个中断源，这个中断源就是硬件提供的，所以中断处理流程中硬件的处理流程也至关重要。由于涉及到硬件，不同架构会有不同的处理规范。在龙芯架构下，需要遵循龙芯中断处理的规范。所以本章将会首先介绍龙芯架构的例外与中断，然后介绍如何获取龙芯架构的中断信息，以及如何设置中断，这就涉及龙芯架构中与中断相关的寄存器。最后实现中断处理，并用一个简单的时钟中断测试中断处理。

### 4.1 龙芯架构的例外与中断概述

首先区分一下例外与中断，例外来自处理器内部，中断来自处理器外部。区分了不同点，也理解一下共同点：例外和中断都是中断源，都可以打断当前的执行流。由于它们有所共同点，所以它们的处理流程有共同的部分，但是又因为它们还是有所不同的，所以在处理流程中要加以区分。

龙芯定义的部分例外有如下内容，详情参阅龙芯架构手册7.4.5节的表7-8。下表和寄存器相关的内容有个印象即可，后续会详细介绍，可返回再看。

<table><thead><tr><th width="103" align="center">Ecode</th><th width="109">Esubcode</th><th width="125">例外代号</th><th>例外类型</th></tr></thead><tbody><tr><td align="center">0</td><td></td><td>INT</td><td>仅当 CSR.ECFG.VS=0 时，表示是中断</td></tr><tr><td align="center">1</td><td></td><td>PIL</td><td>load 操作页无效例外</td></tr><tr><td align="center">2</td><td></td><td>PIS</td><td>store 操作页无效例外</td></tr><tr><td align="center">8</td><td><ul><li>0</li><li>1</li></ul></td><td><ul><li>ADEF</li><li>ADEM</li></ul></td><td><ul><li>取址地址错例外</li><li>访存指令地址错例外</li></ul></td></tr></tbody></table>

中断来自处理器外部，中断信息需要传输到处理器内部，即传输到处理器内部的中断状态位。每个架构有自己不同的实现方式，但是底层逻辑大致都为这样。龙芯架构的中断被处理器硬件标记到指令上以后就被当作一种例外进行处理，因此中断处理遵循普通例外的一部分规则。

了解龙芯的例外与中断之后，介绍一下龙芯例外与中断相关的控制状态寄存器，然后介绍龙芯架构例外处理流程。中断处理流程较例外处理流程多了中断信息采集的步骤，这里中断信息的采集是站在处理器的角度看的。

#### 4.1.1 当前模式信息（CRMD）

该寄存器中的信息用于决定处理器核当前所处的特权等级、全局中断使能、监视点使能和地址翻译模式。

> 此处仅介绍和例外相关的内容。

<table><thead><tr><th width="97" align="center">位</th><th width="77" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">1:0</td><td align="center">PLV</td><td align="center">RW</td><td><p>当前特权等级。</p><p>当触发例外时，硬件将该域的值置为 0，以确保陷入后处于最高特权等级。</p><p>例外返回时，硬件将 CSR.PRMD 的 PPLV 域的值恢复到这里。</p><p>例外返回时，TLB重填例外、机器错误例外对该域的处理不同。</p></td></tr><tr><td align="center">2</td><td align="center">IE</td><td align="center">RW</td><td><p>当前全局中断使能，高有效。当触发例外时，硬件将该域的值置为 0，以确保陷入后屏蔽中断。例外处理程序决定重新开启中断响应时，需显式地将该位置 1。</p><p>例外返回时，硬件将 CSR.PRMD 的 PIE 域的值恢复到这里。</p><p>例外返回时，TLB重填例外、机器错误例外对该域的处理不同。</p></td></tr></tbody></table>

#### 4.1.2 例外前模式信息（PRMD）

当触发例外时，如果例外类型不是 TLB 重填例外和机器错误例外，硬件会将此时处理器核的特权等级、全局中断使能和监视点使能位保存至例外前模式信息寄存器中，用于例外返回时恢复处理器核的现场。

> 此处仅介绍和例外相关的内容。

<table><thead><tr><th width="97" align="center">位</th><th width="77" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">1:0</td><td align="center">PPLV</td><td align="center">RW</td><td>当触发例外时，如果例外类型不是 TLB 重填例外和机器错误例外，硬件会将 CSR.CRMD 中 PLV 域的旧值记录在这个域。</td></tr><tr><td align="center">2</td><td align="center">PIE</td><td align="center">RW</td><td>当触发例外时，如果例外类型不是 TLB 重填例外和机器错误例外，硬件会将 CSR.CRMD 中 IE 域的旧值记录在这个域。</td></tr></tbody></table>

#### 4.1.1 例外配置寄存器（ECFG）

该寄存器用于控制例外和中断的入口计算方式以及各中断的局部使能位。

<table><thead><tr><th width="97" align="center">位</th><th width="79" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">12:0</td><td align="center">LIE</td><td align="center">RW</td><td>局部中断使能位，高有效。这些局部中断使能位与 CSR.ESTAT 中 IS 域记录的 13 个中断源一一对应，每一位控制一个中断源。</td></tr><tr><td align="center">15:13</td><td align="center">0</td><td align="center">R0</td><td>保留域。读返回 0，且软件不允许改变其值。</td></tr><tr><td align="center">18:16</td><td align="center">VS</td><td align="center">RW</td><td>配置例外和中断入口的间距。当 <strong>VS=0</strong> 时，所有例外和中断的入口地址是同一个。当 <strong>VS!=0</strong> 时，各例外和中断之间的入口地址间距是 2^VS条指令。因为 TLB 重填例外和机器错误例外有独立的入口基址，所以二者的例外入口不受 VS 域的影响。</td></tr><tr><td align="center">31:19</td><td align="center">0</td><td align="center">R0</td><td>保留域。读返回 0，且软件不允许改变其值。</td></tr></tbody></table>

#### 4.1.2 例外状态（ESTAT）

<table><thead><tr><th width="97" align="center">位</th><th width="129" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">1:0</td><td align="center">IS[1:0]</td><td align="center">RW</td><td>两个软件中断的状态位。比特 0 和 1 分别对应 SWI0 和 SWI1。 软件中断的设置也是通过这两位完成，软件写 1 置中断写 0 清中断。</td></tr><tr><td align="center">12:2</td><td align="center">IS[12:2]</td><td align="center">R</td><td>中断状态位。其值为 1 表示对应的中断置起。1 个核间中断（IPI），1 个定时器中断（TI），1 个性能计数器溢出中断（PMI），8 个硬中断（HWI0~HWI7）在线中断模式下，硬件仅是逐拍采样各个中断源并将其状态记录与此。此时对于所有中断须为电平中断的要求，是由中断源负责保证，并不在此处维护。</td></tr><tr><td align="center">15:13</td><td align="center">0</td><td align="center">R0</td><td>保留域。读返回 0，且软件不允许改变其值。</td></tr><tr><td align="center">21:16</td><td align="center">Ecode</td><td align="center">R</td><td>例外类型<strong>一级编码</strong>。触发例外时：如果是 TLB 重填例外或机器错误例外，该域保持不变；否则，硬件会根据例外类型将龙芯架构参考手册表 7- 8 中 Ecode 栏定义的数值写入该域。</td></tr><tr><td align="center">30:22</td><td align="center">EsubCode</td><td align="center">R</td><td>例外类型<strong>二级编码</strong>。触发例外时：如果是 TLB 重填例外或机器错误例外，该域保持不变；否则，硬件会根据例外类型将龙芯架构参考手册表 7- 8 中 EsubCode 栏定义的数值写入该域。</td></tr><tr><td align="center">31</td><td align="center">0</td><td align="center">R0</td><td>保留域。读返回 0，且软件不允许改变其值。</td></tr></tbody></table>

#### 4.1.3 例外程序返回地址（ERA）

该寄存器记录普通例外处理完毕之后的返回地址。当触发例外时，如果例外类型既不是 TLB 重填例外也不是机器错误例外，则触发例外的指令的 PC 将被记录在该寄存器中。

<table><thead><tr><th width="131" align="center">位</th><th width="79" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">GRLEN-1:0</td><td align="center">PC</td><td align="center">RW</td><td><p>触发例外时：</p><p>如果是 TLB 重填例外或机器错误例外，该域保持不变；</p><p>否则，硬件会将触发例外的指令的 PC 记录到这里。对于 LA64 架构，在这种情况下，如果触发例外的特权等级处于 32 位地址模式，那么记录的 PC 值的高 32 位强制置为 0。</p></td></tr></tbody></table>

#### 4.1.4 例外入口地址（EENTRY）

该寄存器用于配置普通例外和中断的入口地址。

<table><thead><tr><th width="138" align="center">位</th><th width="79" align="center">名字</th><th width="80" align="center">读写</th><th>描述</th></tr></thead><tbody><tr><td align="center">11:0</td><td align="center">0</td><td align="center">R</td><td>只读恒为 0，写被忽略。</td></tr><tr><td align="center">GRLEN-1:12</td><td align="center">PC</td><td align="center">RW</td><td>普通例外和中断入口地址所在页的页号。</td></tr></tbody></table>

### 4.2 龙芯例外和中断处理流程

#### 4.2.1 例外处理流程

* 将 CSR.CRMD 的 PLV、IE 分别存到 CSR.PRMD 的 PPLV、PIE 中，然后将 CSR.CRMD 的 PLV 置为 0，IE 置为 0；
* 对于支持 Watch 功能的实现，还要将 CSR.CRMD 的 WE 存到 CSR.PRMD 的 PWE 中，然后将 CSR.CRMD 的 WE 置为 0；
* 将触发例外指令的 PC 值记录到 CSR.ERA 中；
* 跳转到例外入口处取指。

当软件执行 ERTN 指令从普通例外执行返回时，处理器硬件会完成如下操作：

* 将 CSR.PRMD 中的 PPLV、PIE 值恢复到 CSR.CRMD 的 PLV、IE 中；
* 对于支持 Watch 功能的实现，还要将 CSR.PRMD 中的 PWE 值恢复到 CSR.CRMD 的 WE 中；
* 跳转到 CSR.ERA 所记录的地址处取指。

#### 4.2.1 中断处理流程

各中断源发来的中断信号被处理器采样至 CSR.ESTAT.IS 域中，这些信息与软件配置在 CSR.ECFG.LIE 域中的局部中断使能信息按位与，得到一个 13 位中断向量 int\_vec。当 CSR.CRMD.IE=1 且 int\_vec 不为全 0 时，处理器认为有需要响应的中断，于是从执行的指令流中挑选出一条指令，将其标记上一种特殊的例外，即中断例外。随后处理器硬件的处理过程与普通例外的处理过程一样。

### 4.3 编写例外中断处理程序

#### 4.3.1 例外与中断初始化配置

```c
/* /* os-loongson/os-elephant-dev/kernel/irq.c */
void arch_init_irq(void)
{
        unsigned int ecfg = ( 0U << CSR_ECFG_VS_SHIFT ) | 0x3fcU | (0x1 << 11);
        unsigned long tcfg = 0x10000000UL | (1U << 0) | (1U << 1);

        clear_csr_ecfg(ECFG0_IM);
	clear_csr_estat(ESTATF_IP);

	write_csr_ecfg(ecfg);

	write_csr_tcfg(tcfg);
	write_csr_eentry((unsigned long)trap_entry);   // trap_entry为例外入口程序，见4.3.2
	arch_local_irq_enable();
}
```

此处还要添加访问有关系统控制寄存器的函数。

> 可参考仓库 ch4-3 分支的 include/loongarch.h 文件内容，依次实现 ecfg、estat、tcfg、eentry 系统控制寄存器的访问函数。

`arch_local_irq_enable()` 函数为打开中断响应。

```c
/* /* os-loongson/os-elephant-dev/kernel/irq.c */
static inline void arch_local_irq_enable(void)
{
	uint32_t flags = CSR_CRMD_IE;
	__asm__ __volatile__(
		"csrxchg %[val], %[mask], %[reg]\n\t"
		: [val] "+r" (flags)
		: [mask] "r" (CSR_CRMD_IE), [reg] "i" (LOONGARCH_CSR_CRMD)
		: "memory");
}
```

#### 4.3.2 例外入口程序

```c
/* os-loongson/os-elephant-dev/kernel/trap_entry.S */
.section .text
.globl trap_entry
.align 4
trap_entry:
        // 流出空间保存寄存器
        addi.d $sp, $sp, -256

        // 保存寄存器
        st.d $ra, $sp, 0
        st.d $tp, $sp, 8
        st.d $sp, $sp, 16
        st.d $a0, $sp, 24
        st.d $a1, $sp, 32
        st.d $a2, $sp, 40
        st.d $a3, $sp, 48
        st.d $a4, $sp, 56
        st.d $a5, $sp, 64
        st.d $a6, $sp, 72
        st.d $a7, $sp, 80
        st.d $t0, $sp, 88
        st.d $t1, $sp, 96
        st.d $t2, $sp, 104
        st.d $t3, $sp, 112
        st.d $t4, $sp, 120
        st.d $t5, $sp, 128
        st.d $t6, $sp, 136
        st.d $t7, $sp, 144
        st.d $t8, $sp, 152
        st.d $r21, $sp,160
        st.d $fp, $sp, 168
        st.d $s0, $sp, 176
        st.d $s1, $sp, 184
        st.d $s2, $sp, 192
        st.d $s3, $sp, 200
        st.d $s4, $sp, 208
        st.d $s5, $sp, 216
        st.d $s6, $sp, 224
        st.d $s7, $sp, 232
        st.d $s8, $sp, 240

	// 调用C例外处理程序
        bl trap_handler

        // restore register
        ld.d $ra, $sp, 0
        ld.d $tp, $sp, 8
        ld.d $sp, $sp, 16
        ld.d $a0, $sp, 24
        ld.d $a1, $sp, 32
        ld.d $a2, $sp, 40
        ld.d $a3, $sp, 48
        ld.d $a4, $sp, 56
        ld.d $a5, $sp, 64
        ld.d $a6, $sp, 72
        ld.d $a7, $sp, 80
        ld.d $t0, $sp, 88
        ld.d $t1, $sp, 96
        ld.d $t2, $sp, 104
        ld.d $t3, $sp, 112
        ld.d $t4, $sp, 120
        ld.d $t5, $sp, 128
        ld.d $t6, $sp, 136
        ld.d $t7, $sp, 144
        ld.d $t8, $sp, 152
        ld.d $r21, $sp,160
        ld.d $fp, $sp, 168
        ld.d $s0, $sp, 176
        ld.d $s1, $sp, 184
        ld.d $s2, $sp, 192
        ld.d $s3, $sp, 200
        ld.d $s4, $sp, 208
        ld.d $s5, $sp, 216
        ld.d $s6, $sp, 224
        ld.d $s7, $sp, 232
        ld.d $s8, $sp, 240

        addi.d $sp, $sp, 256

        // 例外返回
        ertn
```

#### 4.3.3 C例外处理程序

```c
/* os-loongson/os-elephant-dev/kernel/irq.c */
void trap_handler(void)
{
	unsigned int estat = read_csr_estat();
	unsigned int ecfg = read_csr_ecfg();
	unsigned long era = read_csr_era();
	unsigned long prmd = read_csr_prmd();

	if((prmd & CSR_PRMD_PPLV) != 0)
		put_str("kerneltrap: not from privilege0");
	if(intr_get() != 0)
		put_str("kerneltrap: interrupts enabled");

	if (estat & ecfg & (0x1 << 11)) {
		timer_interrupt();
	} else if (estat & ecfg) {
		printk("estat %x, ecfg %x\n", estat, ecfg);
		printk("era=%p eentry=%p\n", read_csr_era(), read_csr_eentry());
		while(1);
	}

	write_csr_era(era);
	write_csr_prmd(prmd);
}
```

其中 `timer_interrupt()` 实现了一个简单的时钟中断处理程序，用来测试中断处理。

```c
/* os-loongson/os-elephant-dev/kernel/irq.c */
void timer_interrupt(void)
{
	put_str("timer interrupt\n");
	/* ack */
	write_csr_ticlr(read_csr_ticlr() | (0x1 << 0));
}
```

最后在 `init_all` 函数中调用 `arch_init_irq` 函数初始化中断。

```c
/* os-loongson/os-elephant-dev/kernel/init.c */

#ifdef CONFIG_LOONGARCH64
extern void arch_init_irq(void);
#endif

void init_all()
{
	char str[] = "os-loongson";
	int a = 1, b = 16;
#ifdef CONFIG_LOONGARCH64
	serial_ns16550a_init(9600);
	put_str("hello os-loongson\n");
#endif
	put_str("init_all\n");
	printk("hello %s-%c%d.%d\n", str, 'v', 0, a);
	printk("init_all: 0x%x\n", b);
#ifndef CONFIG_LOONGARCH64
	idt_init();	     // 初始化中断
#else
	arch_init_irq();
#endif
	while(1);
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

