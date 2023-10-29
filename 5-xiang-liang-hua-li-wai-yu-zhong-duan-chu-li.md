# 5 向量化例外与中断处理

上一章编写了一个简单的例外与中断处理流程，在上一章的例外与中断处理流程中，每触发一个例外或者中断就需要将例外配置寄存器和例外状态寄存器读取一次。读取完成之后，还要计算并判断是属于哪一个例外或者中断。龙芯架构中支持向量化的例外与中断处理，为了避免上述问题，本章将编写一个向量化的例外与中断处理流程。

### 5.1 准备工作

在开始编写向量化的例外与中断处理流程之前，首先进行一些准备工作：

* 编写例外与中断上下文结构体，struct pt\_regs，其中描述了例外与中断发生后需要保存的上下文；
* 在访问struct pt\_regs结构体时，为访问方便，编写该结构体成员相对于该结构体首地址的偏移宏；
* 32位与64位的指令会有些不一样，编写一些可移植性更强的宏操作；
* 为编写例外与中断入口程序时方便，编写保存上下文的宏；
* 编写一些宏方便在汇编文件中生成函数；
* 参考项目仓库中的代码，移植`include/loongarch.h`文件。

编写例外与中断上下文结构体：

```c
/* include/pt_regs.h */
#ifndef _PT_REGS_H
#define _PT_REGS_H

/*
 * This struct defines the way the registers are stored on the stack during
 * a system call/exception. If you add a register here, please also add it to
 * regoffset_table[] in arch/loongarch/kernel/ptrace.c.
 */
struct pt_regs {
	/* hardware irq number */
	unsigned long hwirq;
	/* Main processor registers. */
	unsigned long regs[32];

	/* Original syscall arg0. */
	unsigned long orig_a0;

	/* Special CSR registers. */
	unsigned long csr_era;
	unsigned long csr_badvaddr;
	unsigned long csr_crmd;
	unsigned long csr_prmd;
	unsigned long csr_euen;
	unsigned long csr_ecfg;
	unsigned long csr_estat;
	unsigned long __last[];
} __attribute__((aligned (8)));

#endif /* _PT_REGS_H */
```

编写struct pt\_regs结构体成员相对于该结构体首地址的偏移宏：

```c
/* include/asm-offsets.h */
#ifndef _ASM_OFFSETS_H
#define _ASM_OFFSETS_H

/* LoongArch pt_regs offsets. */
#define PT_HWIRQ 0
#define PT_R0 8 /* offsetof(struct pt_regs, regs[0]) */
#define PT_R1 16 /* offsetof(struct pt_regs, regs[1]) */
#define PT_R2 24 /* offsetof(struct pt_regs, regs[2]) */
#define PT_R3 32 /* offsetof(struct pt_regs, regs[3]) */
#define PT_R4 40 /* offsetof(struct pt_regs, regs[4]) */
#define PT_R5 48 /* offsetof(struct pt_regs, regs[5]) */
#define PT_R6 56 /* offsetof(struct pt_regs, regs[6]) */
#define PT_R7 64 /* offsetof(struct pt_regs, regs[7]) */
#define PT_R8 72 /* offsetof(struct pt_regs, regs[8]) */
#define PT_R9 80 /* offsetof(struct pt_regs, regs[9]) */
#define PT_R10 88 /* offsetof(struct pt_regs, regs[10]) */
#define PT_R11 96 /* offsetof(struct pt_regs, regs[11]) */
#define PT_R12 104 /* offsetof(struct pt_regs, regs[12]) */
#define PT_R13 112 /* offsetof(struct pt_regs, regs[13]) */
#define PT_R14 120 /* offsetof(struct pt_regs, regs[14]) */
#define PT_R15 128 /* offsetof(struct pt_regs, regs[15]) */
#define PT_R16 136 /* offsetof(struct pt_regs, regs[16]) */
#define PT_R17 144 /* offsetof(struct pt_regs, regs[17]) */
#define PT_R18 152 /* offsetof(struct pt_regs, regs[18]) */
#define PT_R19 160 /* offsetof(struct pt_regs, regs[19]) */
#define PT_R20 168 /* offsetof(struct pt_regs, regs[20]) */
#define PT_R21 176 /* offsetof(struct pt_regs, regs[21]) */
#define PT_R22 184 /* offsetof(struct pt_regs, regs[22]) */
#define PT_R23 192 /* offsetof(struct pt_regs, regs[23]) */
#define PT_R24 200 /* offsetof(struct pt_regs, regs[24]) */
#define PT_R25 208 /* offsetof(struct pt_regs, regs[25]) */
#define PT_R26 216 /* offsetof(struct pt_regs, regs[26]) */
#define PT_R27 224 /* offsetof(struct pt_regs, regs[27]) */
#define PT_R28 232 /* offsetof(struct pt_regs, regs[28]) */
#define PT_R29 240 /* offsetof(struct pt_regs, regs[29]) */
#define PT_R30 248 /* offsetof(struct pt_regs, regs[30]) */
#define PT_R31 256 /* offsetof(struct pt_regs, regs[31]) */
#define PT_CRMD 288 /* offsetof(struct pt_regs, csr_crmd) */
#define PT_PRMD 296 /* offsetof(struct pt_regs, csr_prmd) */
#define PT_EUEN 304 /* offsetof(struct pt_regs, csr_euen) */
#define PT_ECFG 312 /* offsetof(struct pt_regs, csr_ecfg) */
#define PT_ESTAT 320 /* offsetof(struct pt_regs, csr_estat) */
#define PT_ERA 272 /* offsetof(struct pt_regs, csr_era) */
#define PT_BVADDR 280 /* offsetof(struct pt_regs, csr_badvaddr) */
#define PT_ORIG_A0 264 /* offsetof(struct pt_regs, orig_a0) */
#define PT_SIZE 328 /* sizeof(struct pt_regs) */

#endif /* _ASM_OFFSETS_H */
```

编写一些可移植性更强的宏操作：

```c
/* include/asm.h */
#ifndef _ASM_H
#define _ASM_H

/*
 * How to add/sub/load/store/shift C int variables.
 */
#if (__SIZEOF_INT__ == 4)
#define INT_ADD		add.w
#define INT_ADDI	addi.w
#define INT_SUB		sub.w
#define INT_L		ld.w
#define INT_S		st.w
#define INT_SLL		slli.w
#define INT_SLLV	sll.w
#define INT_SRL		srli.w
#define INT_SRLV	srl.w
#define INT_SRA		srai.w
#define INT_SRAV	sra.w
#endif

#if (__SIZEOF_INT__ == 8)
#define INT_ADD		add.d
#define INT_ADDI	addi.d
#define INT_SUB		sub.d
#define INT_L		ld.d
#define INT_S		st.d
#define INT_SLL		slli.d
#define INT_SLLV	sll.d
#define INT_SRL		srli.d
#define INT_SRLV	srl.d
#define INT_SRA		srai.d
#define INT_SRAV	sra.d
#endif

/*
 * How to add/sub/load/store/shift C long variables.
 */
#if (__SIZEOF_LONG__ == 4)
#define LONG_ADD	add.w
#define LONG_ADDI	addi.w
#define LONG_SUB	sub.w
#define LONG_L		ld.w
#define LONG_S		st.w
#define LONG_SLL	slli.w
#define LONG_SLLV	sll.w
#define LONG_SRL	srli.w
#define LONG_SRLV	srl.w
#define LONG_SRA	srai.w
#define LONG_SRAV	sra.w

#ifdef __ASSEMBLY__
#define LONG		.word
#endif
#define LONGSIZE	4
#define LONGMASK	3
#define LONGLOG		2
#endif

#if (__SIZEOF_LONG__ == 8)
#define LONG_ADD	add.d
#define LONG_ADDI	addi.d
#define LONG_SUB	sub.d
#define LONG_L		ld.d
#define LONG_S		st.d
#define LONG_SLL	slli.d
#define LONG_SLLV	sll.d
#define LONG_SRL	srli.d
#define LONG_SRLV	srl.d
#define LONG_SRA	srai.d
#define LONG_SRAV	sra.d

#ifdef __ASSEMBLY__
#define LONG		.dword
#endif
#define LONGSIZE	8
#define LONGMASK	7
#define LONGLOG		3
#endif

/*
 * How to add/sub/load/store/shift pointers.
 */
#if (__SIZEOF_POINTER__ == 4)
#define PTR_ADD		add.w
#define PTR_ADDI	addi.w
#define PTR_SUB		sub.w
#define PTR_L		ld.w
#define PTR_S		st.w
#define PTR_LI		li.w
#define PTR_SLL		slli.w
#define PTR_SLLV	sll.w
#define PTR_SRL		srli.w
#define PTR_SRLV	srl.w
#define PTR_SRA		srai.w
#define PTR_SRAV	sra.w

#define PTR_SCALESHIFT	2

#ifdef __ASSEMBLY__
#define PTR		.word
#endif
#define PTRSIZE		4
#define PTRLOG		2
#endif

#if (__SIZEOF_POINTER__ == 8)
#define PTR_ADD		add.d
#define PTR_ADDI	addi.d
#define PTR_SUB		sub.d
#define PTR_L		ld.d
#define PTR_S		st.d
#define PTR_LI		li.d
#define PTR_SLL		slli.d
#define PTR_SLLV	sll.d
#define PTR_SRL		srli.d
#define PTR_SRLV	srl.d
#define PTR_SRA		srai.d
#define PTR_SRAV	sra.d

#define PTR_SCALESHIFT	3

#ifdef __ASSEMBLY__
#define PTR		.dword
#endif
#define PTRSIZE		8
#define PTRLOG		3
#endif

#endif /* _ASM_H */
```

编写保存上下文的宏：

```c
/* include/stackframe.h */
#ifndef _STACKFRAME_H
#define _STACKFRAME_H

#include <loongarch.h>
#include <regdef.h>
#include <asm.h>
#include <asm-offsets.h>

/* Make the addition of cfi info a little easier. */
	.macro cfi_rel_offset reg offset=0 docfi=0
	.if \docfi
	.cfi_rel_offset \reg, \offset
	.endif
	.endm

	.macro cfi_st reg offset=0 docfi=0
	cfi_rel_offset \reg, \offset, \docfi
	LONG_S	\reg, sp, \offset
	.endm

	.macro cfi_restore reg offset=0 docfi=0
	.if \docfi
	.cfi_restore \reg
	.endif
	.endm

	.macro cfi_ld reg offset=0 docfi=0
	LONG_L	\reg, sp, \offset
	cfi_restore \reg \offset \docfi
	.endm

	.macro BACKUP_T0T1
	csrwr	t0, EXCEPTION_KS0
	csrwr	t1, EXCEPTION_KS1
	.endm

	.macro RELOAD_T0T1
	csrrd   t0, EXCEPTION_KS0
	csrrd   t1, EXCEPTION_KS1
	.endm

	.macro	SAVE_TEMP docfi=0
	RELOAD_T0T1
	cfi_st	t0, PT_R12, \docfi
	cfi_st	t1, PT_R13, \docfi
	cfi_st	t2, PT_R14, \docfi
	cfi_st	t3, PT_R15, \docfi
	cfi_st	t4, PT_R16, \docfi
	cfi_st	t5, PT_R17, \docfi
	cfi_st	t6, PT_R18, \docfi
	cfi_st	t7, PT_R19, \docfi
	cfi_st	t8, PT_R20, \docfi
	.endm

	.macro	SAVE_STATIC docfi=0
	cfi_st	s0, PT_R23, \docfi
	cfi_st	s1, PT_R24, \docfi
	cfi_st	s2, PT_R25, \docfi
	cfi_st	s3, PT_R26, \docfi
	cfi_st	s4, PT_R27, \docfi
	cfi_st	s5, PT_R28, \docfi
	cfi_st	s6, PT_R29, \docfi
	cfi_st	s7, PT_R30, \docfi
	cfi_st	s8, PT_R31, \docfi
	.endm

	.macro	SAVE_SOME docfi=0
	csrrd	t1, LOONGARCH_CSR_PRMD
	andi	t1, t1, 0x3	/* extract pplv bit */
	move	t0, sp
	beqz	t1, 8f
	/* Called from user mode, new stack. */
	/* get_saved_sp docfi=\docfi */
8:
	PTR_ADDI sp, sp, -PT_SIZE
	.if \docfi
	.cfi_def_cfa sp, 0
	.endif
	cfi_st	t0, PT_R3, \docfi
	cfi_rel_offset  sp, PT_R3, \docfi
	LONG_S	zero, sp, PT_R0
	csrrd	t0, LOONGARCH_CSR_PRMD
	LONG_S	t0, sp, PT_PRMD
	csrrd	t0, LOONGARCH_CSR_CRMD
	LONG_S	t0, sp, PT_CRMD
	csrrd	t0, LOONGARCH_CSR_EUEN
	LONG_S  t0, sp, PT_EUEN
	csrrd	t0, LOONGARCH_CSR_ECFG
	LONG_S	t0, sp, PT_ECFG
	csrrd	t0, LOONGARCH_CSR_ESTAT
	PTR_S	t0, sp, PT_ESTAT
	cfi_st	ra, PT_R1, \docfi
	cfi_st	a0, PT_R4, \docfi
	cfi_st	a1, PT_R5, \docfi
	cfi_st	a2, PT_R6, \docfi
	cfi_st	a3, PT_R7, \docfi
	cfi_st	a4, PT_R8, \docfi
	cfi_st	a5, PT_R9, \docfi
	cfi_st	a6, PT_R10, \docfi
	cfi_st	a7, PT_R11, \docfi
	csrrd	ra, LOONGARCH_CSR_ERA
	LONG_S	ra, sp, PT_ERA
	.if \docfi
	.cfi_rel_offset ra, PT_ERA
	.endif
	cfi_st	tp, PT_R2, \docfi
	cfi_st	fp, PT_R22, \docfi

	/* Set thread_info if we're coming from user mode */
	csrrd	t0, LOONGARCH_CSR_PRMD
	andi	t0, t0, 0x3	/* extract pplv bit */
	beqz	t0, 9f

	/* li.d	tp, ~_THREAD_MASK */
	/* and	tp, tp, sp */
	/* cfi_st  u0, PT_R21, \docfi */
	/* csrrd	u0, PERCPU_BASE_KS */
9:
	.endm

	.macro	SAVE_ALL docfi=0
	SAVE_SOME \docfi
	SAVE_TEMP \docfi
	SAVE_STATIC \docfi
	.endm

	.macro	RESTORE_TEMP docfi=0
	cfi_ld	t0, PT_R12, \docfi
	cfi_ld	t1, PT_R13, \docfi
	cfi_ld	t2, PT_R14, \docfi
	cfi_ld	t3, PT_R15, \docfi
	cfi_ld	t4, PT_R16, \docfi
	cfi_ld	t5, PT_R17, \docfi
	cfi_ld	t6, PT_R18, \docfi
	cfi_ld	t7, PT_R19, \docfi
	cfi_ld	t8, PT_R20, \docfi
	.endm

	.macro	RESTORE_STATIC docfi=0
	cfi_ld	s0, PT_R23, \docfi
	cfi_ld	s1, PT_R24, \docfi
	cfi_ld	s2, PT_R25, \docfi
	cfi_ld	s3, PT_R26, \docfi
	cfi_ld	s4, PT_R27, \docfi
	cfi_ld	s5, PT_R28, \docfi
	cfi_ld	s6, PT_R29, \docfi
	cfi_ld	s7, PT_R30, \docfi
	cfi_ld	s8, PT_R31, \docfi
	.endm

	.macro	RESTORE_SOME docfi=0
	LONG_L	a0, sp, PT_PRMD
	andi    a0, a0, 0x3	/* extract pplv bit */
	beqz    a0, 8f
	/* cfi_ld  u0, PT_R21, \docfi */
8:
	LONG_L	a0, sp, PT_ERA
	csrwr	a0, LOONGARCH_CSR_ERA
	LONG_L	a0, sp, PT_PRMD
	csrwr	a0, LOONGARCH_CSR_PRMD
	cfi_ld	ra, PT_R1, \docfi
	cfi_ld	a0, PT_R4, \docfi
	cfi_ld	a1, PT_R5, \docfi
	cfi_ld	a2, PT_R6, \docfi
	cfi_ld	a3, PT_R7, \docfi
	cfi_ld	a4, PT_R8, \docfi
	cfi_ld	a5, PT_R9, \docfi
	cfi_ld	a6, PT_R10, \docfi
	cfi_ld	a7, PT_R11, \docfi
	cfi_ld	tp, PT_R2, \docfi
	cfi_ld	fp, PT_R22, \docfi
	.endm

	.macro	RESTORE_SP_AND_RET docfi=0
	cfi_ld	sp, PT_R3, \docfi
	ertn
	.endm

	.macro	RESTORE_ALL_AND_RET docfi=0
	RESTORE_STATIC \docfi
	RESTORE_TEMP \docfi
	RESTORE_SOME \docfi
	RESTORE_SP_AND_RET \docfi
	.endm

#endif /* _STACKFRAME_H */
```

编写一些宏方便在汇编文件中生成函数：

```c
/* include/linkage.h */
#ifndef _LINKAGE_H
#define _LINKAGE_H

#define __ALIGN		.align 2
#define __ALIGN_STR	".align 2"

#ifndef __ALIGN
#define __ALIGN		.align 4,0x90
#define __ALIGN_STR	".align 4,0x90"
#endif

#define ALIGN __ALIGN

#define STT_NOTYPE  0
#define STT_OBJECT  1
#define STT_FUNC    2
#define STT_SECTION 3
#define STT_FILE    4
#define STT_COMMON  5
#define STT_TLS     6

/* Some toolchains use other characters (e.g. '`') to mark new line in macro */
#ifndef ASM_NL
#define ASM_NL		 ;
#endif

/* SYM_ENTRY -- use only if you have to for non-paired symbols */
#ifndef SYM_ENTRY
#define SYM_ENTRY(name, linkage, align...)		\
	linkage(name) ASM_NL				\
	align ASM_NL					\
	name:
#endif

/* SYM_T_FUNC -- type used by assembler to mark functions */
#ifndef SYM_T_FUNC
#define SYM_T_FUNC				STT_FUNC
#endif

/* SYM_L_* -- linkage of symbols */
#define SYM_L_GLOBAL(name)			.globl name
#define SYM_L_WEAK(name)			.weak name
#define SYM_L_LOCAL(name)			/* nothing */

/* SYM_A_* -- align the symbol? */
#define SYM_A_ALIGN				ALIGN
#define SYM_A_NONE				/* nothing */

/* SYM_START -- use only if you have to */
#ifndef SYM_START
#define SYM_START(name, linkage, align...)		\
	SYM_ENTRY(name, linkage, align)
#endif

/* SYM_END -- use only if you have to */
#ifndef SYM_END
#define SYM_END(name, sym_type)				\
	.type name sym_type ASM_NL			\
	.size name, .-name
#endif

#define SYM_FUNC_START(name)				\
	SYM_START(name, SYM_L_GLOBAL, SYM_A_ALIGN)	\
	.cfi_startproc;

#define SYM_FUNC_START_NOALIGN(name)			\
	SYM_START(name, SYM_L_GLOBAL, SYM_A_NONE)	\
	.cfi_startproc;

#define SYM_FUNC_START_LOCAL(name)			\
	SYM_START(name, SYM_L_LOCAL, SYM_A_ALIGN)	\
	.cfi_startproc;

#define SYM_FUNC_START_LOCAL_NOALIGN(name)		\
	SYM_START(name, SYM_L_LOCAL, SYM_A_NONE)	\
	.cfi_startproc;

#define SYM_FUNC_START_WEAK(name)			\
	SYM_START(name, SYM_L_WEAK, SYM_A_ALIGN)	\
	.cfi_startproc;

#define SYM_FUNC_START_WEAK_NOALIGN(name)		\
	SYM_START(name, SYM_L_WEAK, SYM_A_NONE)		\
	.cfi_startproc;

#define SYM_FUNC_END(name)				\
	.cfi_endproc;					\
	SYM_END(name, SYM_T_FUNC)

#endif /* _LINKAGE_H */
```

移植`include/loongarch.h`文件：根据项目仓库的main分支，复制即可。该文件中存在一些配置宏，配置通过编译选项设置，所以修改Makefile文件，如下所示：

```c
/* Makefile */
CFLAGS = -O1 -g -DCONFIG_LOONGARCH64 -DCONFIG_VA_BITS_40 -DCONFIG_PAGE_SIZE_4KB -DCONFIG_PGTABLE_LEVELS=3 -Wall -march=loongarch64 -mabi=lp64s -ffreestanding -fno-common -nostdlib -fno-stack-protector -fno-pie -no-pie -c -I ./include/ -I ./lib/ -I ./lib/kernel/ -I kernel/
AFLAGS = -g -DCONFIG_LOONGARCH64 -DCONFIG_VA_BITS_40 -DCONFIG_PAGE_SIZE_4KB -DCONFIG_PGTABLE_LEVELS=3 -nostdinc -D__ASSEMBLY__ -fno-PIE -march=loongarch64 -mabi=lp64s -G0 -pipe -msoft-float -mexplicit-relocs -ffreestanding -mno-check-zero-division -c -I ./include/
```

### 5.2 编写向量化的例外与中断处理程序

编写生成中断处理入口程序的宏，do\_vint函数后续实现：

```c
/* loongarch/genex.S */
#include <stackframe.h>
#include <linkage.h>

	.macro	BUILD_VI_HANDLER num
	.align	5
SYM_FUNC_START(handle_vint_\num)
	BACKUP_T0T1
	SAVE_ALL
	addi.d	t0, zero, \num
	cfi_st	t0, PT_HWIRQ, 0
	move	a0, sp
	move	a1, sp
	la.abs	t0, do_vint
	jirl	ra, t0, 0
	RESTORE_ALL_AND_RET
SYM_FUNC_END(handle_vint_\num)
	.endm
```

生成中断处理入口程序，并生成中断处理入口程序向量数组：

```c
/* loongarch/genex.S */
	BUILD_VI_HANDLER 0
	BUILD_VI_HANDLER 1
	BUILD_VI_HANDLER 2
	BUILD_VI_HANDLER 3
	BUILD_VI_HANDLER 4
	BUILD_VI_HANDLER 5
	BUILD_VI_HANDLER 6
	BUILD_VI_HANDLER 7
	BUILD_VI_HANDLER 8
	BUILD_VI_HANDLER 9
	BUILD_VI_HANDLER 10
	BUILD_VI_HANDLER 11
	BUILD_VI_HANDLER 12
	BUILD_VI_HANDLER 13

.section .data, "aw"
	.align	3
        .globl  vector_table
vector_table:
	PTR	handle_vint_0
	PTR	handle_vint_1
	PTR	handle_vint_2
	PTR	handle_vint_3
	PTR	handle_vint_4
	PTR	handle_vint_5
	PTR	handle_vint_6
	PTR	handle_vint_7
	PTR	handle_vint_8
	PTR	handle_vint_9
	PTR	handle_vint_10
	PTR	handle_vint_11
	PTR	handle_vint_12
	PTR	handle_vint_13
```

然后将该文件写入Makefile，该文件是汇编文件，编译选项使用AFLAGS。

```makefile
# Makefile
$(BUILD_DIR)/genex.o: loongarch/genex.S
	$(CC) $(AFLAGS) $< -o $@
```

为了能够将本项目的代码方便移植到其他项目，当体系结构相关文件放在 os-elephant-dev/loongarch/ 目录下，与龙芯架构相关的头文件都在 include/ 目录下。将setup.h文件移到 os-elephant-dev/loongarch/ 目录下，并在`setup_arch`函数中写上例外与中断初始化的接口。

移动后修改Makefile中编译setup.o时的依赖。

```c
/* include/setup.h */
#ifndef _SETUP_H
#define _SETUP_H

#define VECSIZE	0x200

extern void set_handler(unsigned long offset, void *addr, unsigned long len);

extern void per_cpu_trap_init(int cpu);
extern void setup_arch(void);
extern void trap_init(void);

#endif /* _SETUP_H */
```

```c
/* loongarch/setup.c */
#include <setup.h>
#include <bootinfo.h>

unsigned long fw_arg0, fw_arg1, fw_arg2;
unsigned long kernelsp;

void setup_arch(void)
{
	/**
	 * 例外与中断的初始化
	 */
	per_cpu_trap_init(0);
}
```

```makefile
# Makefile
$(BUILD_DIR)/setup.o: loongarch/setup.c
	$(CC) $(CFLAGS) $< -o $@
```

per\_cpu\_trap\_init()函数实现如下所示：

```c
/* loongarch/trap.c */
#include <setup.h>
#include <loongarch.h>
#include <pt_regs.h>
#include <cacheflush.h>
#include <stdio-kernel.h>
#include <string.h>

#define SZ_64K		0x00010000

/**
 * 普通例外与中断处理程序入口
 */
unsigned long eentry;
/**
 * tlb重填例外处理程序入口
 */
unsigned long tlbrentry;

long exception_handlers[VECSIZE * 128 / sizeof(long)] __attribute__((aligned(SZ_64K)));

/**
 * configure_exception_vector - 配置例外与中断处理程序入口
 */
static void configure_exception_vector(void)
{
	eentry = (unsigned long)exception_handlers;
	tlbrentry = (unsigned long)exception_handlers + 80 * VECSIZE;

	/**
	 * 设置普通例外与中断处理程序入口
	 */
	csr_write64(eentry, LOONGARCH_CSR_EENTRY);
	/**
	 * 设置机器错误例外处理程序入口
	 */
	csr_write64(eentry, LOONGARCH_CSR_MERRENTRY);
	/**
	 * 设置tlb重填例外处理程序入口
	 */
	csr_write64(tlbrentry, LOONGARCH_CSR_TLBRENTRY);
}

void per_cpu_trap_init(int cpu)
{
	/**
	 * 设置例外与中断处理程序的间距
	 */
	setup_vint_size(VECSIZE);
	/**
	 * 配置例外与中断处理程序入口
	 */
	configure_exception_vector();
}
```

loongarch/trap.c文件是一个新增文件，添加到Makefile中。在包含头文件时还包含了cacheflush.h文件，该文件的实现如下所示：

```c
/* include/cacheflush.h */
#ifndef _CACHEFLUSH_H
#define _CACHEFLUSH_H

void local_flush_icache_range(unsigned long start, unsigned long end);

#endif /* _CACHEFLUSH_H */
```

```c
/* loongarch/cache.c */
#include <cacheflush.h>

/* Cache operations. */
void local_flush_icache_range(unsigned long start, unsigned long end)
{
	asm volatile ("\tibar 0\n"::);
}
```

编写例外与中断处理入口程序C语言部分，注意这里修改了`do_irq()`函数的接口，后面会调整一些体系结构无关代码：

```c
/* loongarch/trap.c */
extern void do_irq(struct pt_regs *regs, uint64_t virq);

/**
 * hwirq_to_virq - 硬件中断号转换为虚拟中断号
 * @hwirq: 硬件中断号
 */
static unsigned long hwirq_to_virq(unsigned long hwirq)
{
	return EXCCODE_INT_START + hwirq;
}

/**
 * do_vint - 中断处理入口程序（C语言部分）
 * @regs: 指向中断栈内容
 * @sp: 中断栈指针
 * regs == sp
 */
void do_vint(struct pt_regs *regs, unsigned long sp)
{
	unsigned long hwirq = *(unsigned long *)regs;
	unsigned long virq;

	virq = hwirq_to_virq(hwirq);
	do_irq(regs, virq);
}
```

编写例外与中断处理程序初始化：

```c
/* loongarch/trap.c */
extern void *vector_table[];

/**
 * set_handler - 设置例外处理程序句柄
 *
 * @offset: 相对于例外处理程序入口地址的偏移量
 * @addr: 例外处理程序地址
 * @size: 例外处理程序大小
 */
void set_handler(unsigned long offset, void *addr, unsigned long size)
{
	memcpy((void *)eentry + offset, addr, size);
	local_flush_icache_range(eentry + offset, eentry + offset + size);
}

/**
 * trap_init - 例外与中断处理初始化
 */
void trap_init(void)
{
	unsigned long i;
	void *vector_start;
	unsigned long tcfg = 0x01000000UL | (1U << 0) | (1U << 1);
	unsigned long ecfg;

	/**
	 * 清空中断状态
	 */
	clear_csr_ecfg(ECFG0_IM);
	clear_csr_estat(ESTATF_IP);

	/**
	 * 初始化中断处理入口程序
	 */
	for (i = EXCCODE_INT_START; i < EXCCODE_INT_END; i++) {
		vector_start = vector_table[i - EXCCODE_INT_START];
		set_handler(i * VECSIZE, vector_start, VECSIZE);
	}

	local_flush_icache_range(eentry, eentry + 0x400);

	write_csr_tcfg(tcfg);
	ecfg = read_csr_ecfg();
	change_csr_ecfg(CSR_ECFG_IM, ecfg | 0x1 << 11);
}
```

接下来，调整体系结构无关代码，现在例外与中断处理函数的接口变了，所以修改一下函数指针声明。为了不影响x86架构的代码，使用条件编译避免修改x86架构的指针声明。并且将体系结构无关代码的初始化函数换一个函数名，即`irq_init()`函数：

```c
/* kernel/interrupt.h */
#ifndef CONFIG_LOONGARCH64

typedef void* intr_handler;
void idt_init(void);

#else

#include <pt_regs.h>

typedef void (*intr_handler)(struct pt_regs *regs);
void irq_init(void);

#endif
```

修改体系结构无关代码中，中断处理程序初始化函数：

```c
/* kernel/irq.c */
void irq_init(void)
{
	exception_init();
	register_handler(EXCCODE_TIMER, timer_interrupt);

	printk("irq_init done\n");
}
```

修改`do_irq()`函数：

```c
/* kernel/irq.c */
void do_irq(struct pt_regs *regs, uint64_t virq)
{
	// printk("virq = %d ", virq);
	intr_table[virq](regs);
}
```

修改通用中断处理程序：

```c
/* kernel/irq.c */
static void general_intr_handler(struct pt_regs *regs)
{
	unsigned long hwirq = *(unsigned long *)regs;
	unsigned long virq = EXCCODE_INT_START + hwirq;
	printk("!!!!!!!      exception message begin  !!!!!!!!\n");
	printk("intr_table[%d]: %s happened", intr_name[virq]);
	printk("\n!!!!!!!      exception message end    !!!!!!!!\n");
	while(1);
}
```

修改临时的时钟中断处理程序：

```c
/* kernel/irq.c */
void timer_interrupt(struct pt_regs *regs)
{
	printk("timer interrupt\n");
	/* ack */
	write_csr_ticlr(read_csr_ticlr() | (0x1 << 0));
}
```

删除`trap_handler()`函数、`trap_entry.S`文件。删除trap\_entry.S文件之后，在`Makfile`中也删除`$(BUILD_DIR)/trap_entry.o`编译目标。最后在`init_all()`函数中进行测试：

```c
/* kernel/init.c */
#ifdef CONFIG_LOONGARCH64
#include <setup.h>
#include <ns16550a.h>
#endif

#ifdef CONFIG_LOONGARCH64
extern void irq_init(void);
#endif

/*负责初始化所有模块 */
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
	setup_arch();
	trap_init();
	irq_init();
	intr_enable();
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
