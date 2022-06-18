---
title: RISC-V Syscall 系列2：Syscall 过程分析
subtitle: 

# Summary for listings and search engines
summary: Syscall 的执行过程是怎样的？

# Link this post with a project
projects: []

# Date published
date: '2022-06-18T00:00:00Z'

# Date updated
lastmod: '2022-06-18T11:30:00Z'

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ''
  placement: 2
  preview_only: false

authors:
  - admin

tags:
  - Linux
  - Syscall
  - RISC-V

categories:
  - 技术
---


## Syscall 总体流程

![syscall_procedure](http://assets.processon.com/chart_image/62ab2ebbe0b34d29447a9392.png)

大体分为以下5个过程：
    - 用户态程序执行 ecall 指令触发 trap，进入内核态
    - 执行 trap 进入过程
    - 执行实际系统调用函数
    - 执行 trap 退出过程
    - 执行 sret 指令返回用户态继续执行


## ecall


### 相关特权寄存器简介

#### stvec (Supervisor Trap Vector Base Address Register) 用户保存发送异常时处理器需要跳转到的地址

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2F3YGvjmyxgo.png?alt=media&token=85170e16-0fd0-4eb1-9006-17e95b437517)

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FXEDRWsp_S_.png?alt=media&token=0e2707ee-0143-4bfa-a668-818b24808c32)

根据上图中 RISC-V 规范中的描述，stvec 寄存器分为两部分，低 2 位称为 MODE，其他位称为 BASE。BASE 用于 trap 入口函数的基地址，必须保证四字节对齐。MODE 用于控制入口函数的地址配置方式。

MODE=0 时，表示使用 Direct 方式，exception 发生后 PC 都跳转到 BASE 指定的地址处。
MODE=1 时，表示使用 Vectored 方式，exception 的处理方式同 Direct，但 interrupt 的入口地址以数组方式排列。


#### sepc (Supervisor Exception Program Counter) 当发生 trap 时，处理器会将发生 trap 所对应的指令的地址（pc）保存在 sepc 中

#### scause (Supervisor Cause Register) 当 trap 发生时，处理器会设置该寄存器表示 trap 发生的原因

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FD1I3UDVjRw.png?alt=media&token=ed2f4ff1-7166-4079-94cf-84c34d28211c)

根据 RISC-V 规范，scause 寄存器由高 1 位的 Interrupt 和 其他位的 Exception Code 组成。

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FZmotNm6dAa.png?alt=media&token=fcf94551-558b-4d80-905a-a86fb5705db3)

Syscall 触发时设置的内容如上图中红框所示，Interrupt=0 表示是异常，Exception Code=8，表示是从用户态执行的 ecall。


#### sstatus (Supervisor Status Register) 用于跟踪和控制处理器当前操作状态（比如包括关闭和打开全局中断）

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FScMRkYiy14.png?alt=media&token=88d28c11-4462-4f32-9759-ab467ffc4b16)

根据 RISC-V 规范，UIE、SIE 分别用于打开（1）或者关闭（0）用户/内核模式下的全局中断。UPIE、SPIE 用于当 trap 发生时保存 trap 发生之前的 UIE、SIE 值。SPP 用于当 trap 发生时用于保存 trap 发生之前的权限级别值。

### ecall 指令主要做了以下几件事

- 保存 pc 到 sepc
- 将 cpu 模式由用户态提升到内核态
- 设置 scause 寄存器，表示是 exception，并且 exception code 设置为 8，表示是从用户态执行的 trap
- 跳转到 stvec 指向的异常处理代码


## 进入内核异常处理进入部分

### 异常处理函数入口初始化

操作系统初始化时就将异常处理函数地址设置到了 stvec 寄存器中，具体初始化代码如下

```asm
//
// arch/riscv/kernel/head.S
//
setup_trap_vector:
	/* Set trap vector to exception handler */
	la a0, handle_exception
  	csrw CSR_TVEC, a0    // 将异常处理函数地址设置到 stvec 寄存器


//
// arch/riscv/kernel/entry.S
//
ENTRY(handle_exception)  // 声明异常处理函数入口


//
// include/linux/linkage.h
//
#define ENTRY(name) \
	SYM_FUNC_START(name)
#define SYM_FUNC_START(name)				\
	SYM_START(name, SYM_L_GLOBAL, SYM_A_ALIGN)
#define SYM_A_ALIGN				ALIGN
#define ALIGN __ALIGN


//
// arch/riscv/include/asm/linkage.h
//
#define __ALIGN		.balign 4  // 4字节对齐

```

根据 RISC-V 规范，stvec 由 BASE、MODE 两部分组成。因为汇编代码里将 handle_exception 声明为 4 字节对齐，故 handle_exception 的地址的低 2 位都是 0。所以 stvec 处于 Direct 模式。下图中是实际调试 Linux 时打印的 stvec 和 handle_exception 的地址。

```sh
(gdb) p/x $stvec
$8 = 0xffffffff80002f18
  
(gdb) b handle_exception
Breakpoint 1 at 0xffffffff80002f18: file arch/riscv/kernel/entry.S, line 27.

```

### 上下文保存

当处理器执行完 ecall 指令后，跳转到 handle_exception 处。

```asm
//
// arch/riscv/kernel/entry.S
//
ENTRY(handle_exception)
  // 交换 tp 和 sscratch 寄存器的值，从 kernel 进入时 sscratch=0
  csrrw tp, CSR_SCRATCH, tp
  // tp!=0 时（即从用户态进入时），跳转到 _save_context
  bnez tp, _save_context

...
_save_context:
  // 保存用户线程栈地址
  REG_S sp, TASK_TI_USER_SP(tp)
  // sp 设置为内核线程栈地址
  REG_L sp, TASK_TI_KERNEL_SP(tp) 
  // 分配内核栈空间，以存放用户线程上下文环境（即除 x0 以外的 31 个通用寄存器和 csr 寄存器）
  addi sp, sp, -(PT_SIZE_ON_STACK)
  REG_S x1,  PT_RA(sp)
  REG_S x3,  PT_GP(sp)
  ...
  REG_S x31, PT_T6(sp)
  ...
  REG_S s0, PT_SP(sp)
  REG_S s1, PT_STATUS(sp)
  REG_S s2, PT_EPC(sp)
  REG_S s3, PT_BADADDR(sp)
  REG_S s4, PT_CAUSE(sp)
  REG_S s5, PT_TP(sp)
  // sscratch=0，当出现嵌套异常时，用来判断是从用户态还是内核态进入
  csrw CSR_SCRATCH, x0
  ...
  // 设置函数调用的返回地址
  la ra, ret_from_exception
  // 当前 s4=scause，表示异常码，这里判断异常码为 8 时，表示是系统调用，则跳转到 handle_syscall
  li t0, EXC_SYSCALL
  beq s4, t0, handle_syscall
  
  
//
// include/generated/asm-offsets.h
//
#define TASK_TI_USER_SP 24
#define TASK_TI_KERNEL_SP 16
#define PT_RA 8
#define PT_GP 24
#define PT_T6 248
#define PT_SIZE_ON_STACK 288

//
// arch/riscv/include/ams/csr.h
//
#define EXC_SYSCALL		8
```


## 进入系统调用处理

```asm
//
// arch/riscv/kernel/entry.S
//

handle_syscall:
  ...
  // 将栈上存放 sepc 的值 +4，以便 Syscall 返回到用户态时继续执行 ecall 的下一条指令。这里 sepc+=4 的原因是处理器因异常进入 trap 时，sepc 会被设置成产生异常的指令地址，也就是说当前 sepc 指向了出发系统调用的 ecall 指令地址，而当系统调用返回用户空间时，我们希望是接着执行 ecall 的下一条指令，故需要 +4 来让系统调用执行完后能够返回到正确的指令地址执行。
  addi s2, s2, 0x4
  REG_S s2, PT_EPC(sp)
  ...
  // 接下来是找到对于系统调用处理函数的地址
  // a7是系统调用时传递的系统调用号参数，sys_call_table 是系统调用映射表的地址
  // 下面的代码可以理解为 s0 = sys_call_table[a7]
  la s0, sys_call_table
  slli t0, a7, RISCV_LGPTR
  add s0, s0, t0
  REG_L s0, 0(s0)
  // 跳转到对于系统调用函数
  jalr s0


//
// arch/riscv/include/asm/syscall.h
//
extern void * const sys_call_table[];


//
// arch/riscv/include/asm/asm.h
//
#define REG_S		__REG_SEL(sd, sw)
#define __REG_SEL(a, b)	__ASM_STR(a)
#define __ASM_STR(x)	x
#define RISCV_LGPTR		"3"
```

```asm
//
// arch/riscv/kernel/syscall_table.c
//
#define __SYSCALL(nr, call)	[nr] = (call),

void * const sys_call_table[__NR_syscalls] = {
	[0 ... __NR_syscalls - 1] = sys_ni_syscall,
#include <asm/unistd.h>
};


//
// include/uapi/asm-generic/unistd.h
//
#define __NR_syscalls 451

#define __NR_write 64
__SYSCALL(__NR_write, sys_write)


//
// include/linux/syscalls.h
//
asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count);

#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

#define __SYSCALL_DEFINEx(x, name, ...)					\
	__diag_push();							\
	__diag_ignore(GCC, 8, "-Wattribute-alias",			\
		      "Type aliasing is used to sanitize syscall arguments");\
	asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
		__attribute__((alias(__stringify(__se_sys##name))));	\
	ALLOW_ERROR_INJECTION(sys##name, ERRNO);			\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	__diag_pop();							\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))

#define __MAP(n,...) __MAP##n(__VA_ARGS__)
#define __SC_DECL(t, a)	t a



// fs/read_write.c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	return ksys_write(fd, buf, count);
}

```

## 进入内核异常处理退出部分

```asm
//
// arch/riscv/kernel/entry.S
//

ret_from_syscall:
  // 将系统调用的返回值 a0 更新到用户态线程的上下文中
  REG_S a0, PT_A0(sp)
  ...
  // 恢复内核栈并保存
  addi s0, sp, PT_SIZE_ON_STACK
  REG_S s0, TASK_TI_KERNEL_SP(tp)
  // 恢复用户态线程栈上下文
  REG_L a0, PT_STATUS(sp)
  REG_L  a2, PT_EPC(sp)
  REG_SC x0, a2, PT_EPC(sp)
  csrw CSR_STATUS, a0
  csrw CSR_EPC, a2
  REG_L x1,  PT_RA(sp)
  REG_L x3,  PT_GP(sp)
  ...
  REG_L x31, PT_T6(sp)
  REG_L x2,  PT_SP(sp)
  // 返回用户态
  sret
```

## 参考资料

- https://gitee.com/unicornx/riscv-operating-system-mooc/raw/main/slides/ch10-trap-exception.pdf
- https://jborza.com/emulation/2021/04/22/ecalls-and-syscalls.html
- https://gitee.com/unicornx/riscv-operating-system-mooc/raw/main/slides/ch16-syscall.pdf
- https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf
- https://www.kernel.org/doc/html/v5.17/process/adding-syscalls.html
- https://blog.csdn.net/rikeyone/article/details/91047118
