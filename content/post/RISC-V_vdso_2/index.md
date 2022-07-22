---
title: RISC-V vDSO 系列 4：vDSO 实现原理分析
subtitle: 

# Summary for listings and search engines
summary: 本文通过对 Linux 内核代码的分析，帮助读者了解 vDSO 的实现原理。

# Link this post with a project
projects: []

# Date published
date: '2022-07-20T00:00:00Z'

# Date updated
lastmod: '2022-07-21T11:30:00Z'

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
  - vDSO
  - vsyscall

categories:
  - 技术
---

## 概述

在上一篇文章[什么是 vDSO](../RISC-V_vdso_1/index.md)中介绍了 vDSO 的相关背景和概念，本篇文章会进一步通过对 Linux 内核及 glibc 相关代码的研究，来分析 vDSO 的实现原理。

说明：文中涉及的 Linux 源码是基于 5.17 版本，glibc 是基于 2.35 版本。


## Build

Linux 内核中 vDSO 代码包括以下几部分：
* lib/vdso/：架构无关部分
  * gettimeofday.c
* arch/riscv/kernel/：架构相关部分
  * vdso.c：数据结构定义及初始化
  * vdso/：导出函数入口
    * flush_icache.S
    * getcpu.S
    * rt_sigreturn.S
    * vgettimeofday.c
    * vdso.S
    * vdso.lds.S

> 下面未加路径的文件默认路径为 arch/riscv/kernel/vdso

```mermaid
flowchart LR;
    J(lib/vdso/gettimeofday.c)-->F;
    A(vgettimeofday.c)-->F(vdso.so.dbg / linux-vdso.so.1);
    B(flush_icache.S)-->F;
    C(getcpu.S)-->F;
    D(rt_sigreturn.S)-->F;
    M(note.S)-->F;
    E(vdso.lds.S)-->F;
    F-- objcopy -S --->G(vdso.so)
    G-->H(vdso.o)
    I(vdso.S)-- .incbin --->H
    H-->K(kernel)
    L(arch/riscv/kernel/vdso.c)-->K
```

上图描述了上述代码如何编译成 `linux-vdso.so.1` 及如何集成到内核中的大体流程。整个流程大致可以分为两个阶段：
1. 生成共享库 `linux-vdso.so.1`
2. 共享库集成到内核

下面会结合内核编译日志和内核源码一起分析整个构建过程。

### 生成共享库 `linux-vdso.so.1`

```sh
  riscv64-linux-gnu-gcc -E -Wp,-MMD,arch/riscv/kernel/vdso/.vdso.lds.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./=    -P -C -Uriscv -P -Uriscv -D__ASSEMBLY__ -DLINKER_SCRIPT -o arch/riscv/kernel/vdso/vdso.lds arch/riscv/kernel/vdso/vdso.lds.S
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.rt_sigreturn.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./= -D__ASSEMBLY__ -fno-PIE -mabi=lp64 -march=rv64imafdc -Wa,-gdwarf-2    -c -o arch/riscv/kernel/vdso/rt_sigreturn.o arch/riscv/kernel/vdso/rt_sigreturn.S 
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.vgettimeofday.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mabi=lp64 -march=rv64imac -mno-save-restore -DCONFIG_PAGE_OFFSET=0xffffaf8000000000 -mcmodel=medany -fno-omit-frame-pointer -mstrict-align -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 --param=allow-store-data-races=0 -Wframe-larger-than=2048 -fstack-protector-strong -Wimplicit-fallthrough=5 -Wno-main -Wno-unused-but-set-variable -Wno-unused-const-variable -fno-omit-frame-pointer -fno-optimize-sibling-calls -fno-stack-clash-protection -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wcast-function-type -Wno-stringop-truncation -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -Wno-alloc-size-larger-than -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -Wno-packed-not-aligned -g -fno-stack-protector -fPIC -include /labs/linux-lab/src/linux-stable/lib/vdso/gettimeofday.c    -DKBUILD_MODFILE='"arch/riscv/kernel/vdso/vgettimeofday"' -DKBUILD_BASENAME='"vgettimeofday"' -DKBUILD_MODNAME='"vgettimeofday"' -D__KBUILD_MODNAME=kmod_vgettimeofday -c -o arch/riscv/kernel/vdso/vgettimeofday.o arch/riscv/kernel/vdso/vgettimeofday.c 
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.getcpu.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./= -D__ASSEMBLY__ -fno-PIE -mabi=lp64 -march=rv64imafdc -Wa,-gdwarf-2    -c -o arch/riscv/kernel/vdso/getcpu.o arch/riscv/kernel/vdso/getcpu.S 
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.flush_icache.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./= -D__ASSEMBLY__ -fno-PIE -mabi=lp64 -march=rv64imafdc -Wa,-gdwarf-2    -c -o arch/riscv/kernel/vdso/flush_icache.o arch/riscv/kernel/vdso/flush_icache.S 
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.note.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./= -D__ASSEMBLY__ -fno-PIE -mabi=lp64 -march=rv64imafdc -Wa,-gdwarf-2    -c -o arch/riscv/kernel/vdso/note.o arch/riscv/kernel/vdso/note.S 

```

从上述编译日志可以看出，首先 `vdso.lds.S` 是链接脚本文件，会通过 `gcc -E` 命令执行预处理。然后 `lib/vdso/gettimeofday.c`，`vgettimeofday.c`，`flush_icache.S`，`getcpu.S`，`rt_sigreturn.S`，`note.S` 这几个文件会通过 `gcc -c` 命令编译成 `.o` 文件。

```
riscv64-linux-gnu-ld  -melf64lriscv   -shared -S -soname=linux-vdso.so.1 --build-id=sha1 --hash-style=both --eh-frame-hdr -T arch/riscv/kernel/vdso/vdso.lds arch/riscv/kernel/vdso/rt_sigreturn.o arch/riscv/kernel/vdso/vgettimeofday.o arch/riscv/kernel/vdso/getcpu.o arch/riscv/kernel/vdso/flush_icache.o arch/riscv/kernel/vdso/note.o -o arch/riscv/kernel/vdso/vdso.so.dbg.tmp && riscv64-linux-gnu-objcopy  -G __vdso_rt_sigreturn  -G __vdso_vgettimeofday  -G __vdso_getcpu  -G __vdso_flush_icache arch/riscv/kernel/vdso/vdso.so.dbg.tmp arch/riscv/kernel/vdso/vdso.so.dbg && rm arch/riscv/kernel/vdso/vdso.so.dbg.tmp
```

接下来是生成 `vdso.so.dbg`。就是把上一步生成的中间文件通过 `ld` 命令链接起来生成共享库文件。这里通过 `-soname=linux-vdso.so.1` 参数指定了库的真实名字。另外其中的 `objcopy -G` 命令是将本地函数变为全局函数，我理解现在的版本中已经不需要了，因为在后面的流程中，会移除静态符号表信息。

`vdso.so.dbg` 的真实名字就是 `linux-vdso.so.1`，也可以通过下面的命令进行验证：
```sh
$ readelf -d  /labs/linux-lab/build/riscv64/virt/linux/v5.17/arch/riscv/kernel/vdso/vdso.so.dbg

Dynamic section at offset 0x390 contains 14 entries:
  Tag        Type                         Name/Value
 0x000000000000000e (SONAME)             Library soname: [linux-vdso.so.1]
 0x0000000000000004 (HASH)               0x120
 0x000000006ffffef5 (GNU_HASH)           0x158
 0x0000000000000005 (STRTAB)             0x270
 0x0000000000000006 (SYMTAB)             0x198
 0x000000000000000a (STRSZ)              143 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000007 (RELA)               0x0
 0x0000000000000008 (RELASZ)             0 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffc (VERDEF)             0x318
 0x000000006ffffffd (VERDEFNUM)          2
 0x000000006ffffff0 (VERSYM)             0x300
 0x0000000000000000 (NULL)               0x0
```

### 共享库集成到内核

```sh
riscv64-linux-gnu-objcopy -S  arch/riscv/kernel/vdso/vdso.so.dbg arch/riscv/kernel/vdso/vdso.so
```

先通过 `objcopy -S` 命令将 `vdso.so.dbg` 移除符号信息进而生成 `vdso.so`。这主要是为了减少集成到内核的代码大小。

```sh
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/.vdso.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -fmacro-prefix-map=./= -Wall -Wundef -Werror=strict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -fshort-wchar -fno-PIE -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Wno-format-security -std=gnu89 -mabi=lp64 -march=rv64imac -mno-save-restore -DCONFIG_PAGE_OFFSET=0xffffaf8000000000 -mcmodel=medany -fno-omit-frame-pointer -mstrict-align -fno-delete-null-pointer-checks -Wno-frame-address -Wno-format-truncation -Wno-format-overflow -Wno-address-of-packed-member -O2 --param=allow-store-data-races=0 -Wframe-larger-than=2048 -fstack-protector-strong -Wimplicit-fallthrough=5 -Wno-main -Wno-unused-but-set-variable -Wno-unused-const-variable -fno-omit-frame-pointer -fno-optimize-sibling-calls -fno-stack-clash-protection -Wdeclaration-after-statement -Wvla -Wno-pointer-sign -Wcast-function-type -Wno-stringop-truncation -Wno-array-bounds -Wno-stringop-overflow -Wno-restrict -Wno-maybe-uninitialized -Wno-alloc-size-larger-than -fno-strict-overflow -fno-stack-check -fconserve-stack -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -Wno-packed-not-aligned -g    -DKBUILD_MODFILE='"arch/riscv/kernel/vdso"' -DKBUILD_BASENAME='"vdso"' -DKBUILD_MODNAME='"vdso"' -D__KBUILD_MODNAME=kmod_vdso -c -o arch/riscv/kernel/vdso.o arch/riscv/kernel/vdso.c 

```

然后通过 `gcc` 命令将 `arch/riscv/kernel/vdso.c` 编译成 `arch/riscv/kernel/vdso.o` 文件。

```
  riscv64-linux-gnu-gcc -Wp,-MMD,arch/riscv/kernel/vdso/.vdso.o.d  -nostdinc -I./arch/riscv/include -I./arch/riscv/include/generated  -I./include -I./arch/riscv/include/uapi -I./arch/riscv/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -D__KERNEL__ -fmacro-prefix-map=./= -D__ASSEMBLY__ -fno-PIE -mabi=lp64 -march=rv64imafdc -Wa,-gdwarf-2    -c -o arch/riscv/kernel/vdso/vdso.o arch/riscv/kernel/vdso/vdso.S 

```
然后又通过 `gcc` 命令将 `vdso.S` 编译生成了 `vdso.o` 文件。`vdso.S` 文件内部其实就是通过 `.incbin` 将 `vdso.so` 共享库包含进来，同时设置一下内存页对齐。`vdso.S` 的代码如下：
```asm
#include <linux/init.h>
#include <linux/linkage.h>
#include <asm/page.h>

	__PAGE_ALIGNED_DATA

	.globl vdso_start, vdso_end
	.balign PAGE_SIZE
vdso_start:
	.incbin "arch/riscv/kernel/vdso/vdso.so"
	.balign PAGE_SIZE
vdso_end:

	.previous
```
>注意这里的 `vdso.o` 文件和上一步生成的 `arch/riscv/kernel/vdso.o` 不在同一个目录下。

```sh
  riscv64-linux-gnu-ar cDPrST arch/riscv/kernel/vdso/built-in.a arch/riscv/kernel/vdso/vdso.o
  riscv64-linux-gnu-ar cDPrST arch/riscv/kernel/built-in.a arch/riscv/kernel/vdso.o arch/riscv/kernel/vdso/built-in.a ...
```

然后通过 `ar` 命令将 `vdso.o` 打包到 `built-in.a` 文件中，再将 `built-in.a` 和 `arch/riscv/kernel/vdso.o` 一起打包到 `arch/riscv/kernel/built-in.a` 文件中，最终被打包进内核中。

通过下面的命令验证，能看出 `vdso.so` 移除了静态符号表信息。

```sh
$ readelf -sW /labs/linux-lab/build/riscv64/virt/linux/v5.17/arch/riscv/kernel/vdso/vdso.so.dbg

Symbol table '.dynsym' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000000004e8     0 SECTION LOCAL  DEFAULT   10 
     2: 0000000000000a64   394 FUNC    GLOBAL DEFAULT   11 __vdso_gettimeofday@@LINUX_4.15
     3: 0000000000000bee   122 FUNC    GLOBAL DEFAULT   11 __vdso_clock_getres@@LINUX_4.15
     4: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  ABS LINUX_4.15
     5: 0000000000000800     8 FUNC    GLOBAL DEFAULT   11 __vdso_rt_sigreturn@@LINUX_4.15
     6: 000000000000080a   602 FUNC    GLOBAL DEFAULT   11 __vdso_clock_gettime@@LINUX_4.15
     7: 0000000000000c74    10 FUNC    GLOBAL DEFAULT   11 __vdso_flush_icache@@LINUX_4.15
     8: 0000000000000c68    10 FUNC    GLOBAL DEFAULT   11 __vdso_getcpu@@LINUX_4.15

Symbol table '.symtab' contains 29 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000120     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000000158     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000000198     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000270     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000300     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000318     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000000350     0 SECTION LOCAL  DEFAULT    7 
     8: 0000000000000390     0 SECTION LOCAL  DEFAULT    8 
     9: 00000000000004c0     0 SECTION LOCAL  DEFAULT    9 
    10: 00000000000004e8     0 SECTION LOCAL  DEFAULT   10 
    11: 0000000000000800     0 SECTION LOCAL  DEFAULT   11 
    12: 0000000000000c80     0 SECTION LOCAL  DEFAULT   12 
    13: 0000000000000000     0 SECTION LOCAL  DEFAULT   13 
    14: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS vgettimeofday.c
    15: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    16: fffffffffffff000     0 NOTYPE  LOCAL  DEFAULT  ABS _timens_data
    17: 0000000000000390     0 OBJECT  LOCAL  DEFAULT  ABS _DYNAMIC
    18: 0000000000000c80     0 OBJECT  LOCAL  DEFAULT  ABS _PROCEDURE_LINKAGE_TABLE_
    19: ffffffffffffe000     0 NOTYPE  LOCAL  DEFAULT    1 _vdso_data
    20: 00000000000004c0     0 NOTYPE  LOCAL  DEFAULT    9 __GNU_EH_FRAME_HDR
    21: 0000000000000c80     0 OBJECT  LOCAL  DEFAULT  ABS _GLOBAL_OFFSET_TABLE_
    22: 0000000000000000     0 OBJECT  LOCAL  DEFAULT  ABS LINUX_4.15
    23: 0000000000000a64   394 FUNC    LOCAL  DEFAULT   11 __vdso_gettimeofday
    24: 0000000000000bee   122 FUNC    LOCAL  DEFAULT   11 __vdso_clock_getres
    25: 000000000000080a   602 FUNC    LOCAL  DEFAULT   11 __vdso_clock_gettime
    26: 0000000000000c74    10 FUNC    GLOBAL DEFAULT   11 __vdso_flush_icache
    27: 0000000000000c68    10 FUNC    GLOBAL DEFAULT   11 __vdso_getcpu
    28: 0000000000000800     8 FUNC    GLOBAL DEFAULT   11 __vdso_rt_sigreturn
$ readelf -sW /labs/linux-lab/build/riscv64/virt/linux/v5.17/arch/riscv/kernel/vdso/vdso.so

Symbol table '.dynsym' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000000004e8     0 SECTION LOCAL  DEFAULT   10 
     2: 0000000000000a64   394 FUNC    GLOBAL DEFAULT   11 __vdso_gettimeofday@@LINUX_4.15
     3: 0000000000000bee   122 FUNC    GLOBAL DEFAULT   11 __vdso_clock_getres@@LINUX_4.15
     4: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  ABS LINUX_4.15
     5: 0000000000000800     8 FUNC    GLOBAL DEFAULT   11 __vdso_rt_sigreturn@@LINUX_4.15
     6: 000000000000080a   602 FUNC    GLOBAL DEFAULT   11 __vdso_clock_gettime@@LINUX_4.15
     7: 0000000000000c74    10 FUNC    GLOBAL DEFAULT   11 __vdso_flush_icache@@LINUX_4.15
     8: 0000000000000c68    10 FUNC    GLOBAL DEFAULT   11 __vdso_getcpu@@LINUX_4.15
```



## Architecture




### 内核初始化

### 进程启动设置

### 虚拟系统调用

### 内核更新数据


## 相关 Patch

## kernel and userspace setup

vDSO 初始化 [1]。

![vdso_setup](vdso_setup.png)

fs/exec.c 
    do_execve
        do_execveat_common
            bprm_execve
                exec_binprm
                    search_binary_handler
                        load_elf_binary / fmt->load_binary(bprm)
                            ARCH_SETUP_ADDITIONAL_PAGES
                                arch/riscv/kernel/vdso.c arch_setup_additional_pages
                                    __setup_additional_pages
                                        [vdso] [vvar] 映射到用户内存
                                        _install_special_mapping
                                            VM_READ
                                            VM_EXEC
                                        
                            create_elf_tables
                                ARCH_DLINFO
                                    AT_SYSINFO_EHDR
                                copy_to_user(sp, mm->saved_auxv)
                            START_THREAD / start_thread


arch/riscv/kernel/vdso.c
    arch_initcall(vdso_init);
    vdso_init   初始化 vdso_info 对象
        vvar_fault
        vdso_mremap
        __vdso_init 
            pfn
include/vdso/datapage.h
arch/riscv/include/asm/vdso/vsyscall.h
arch/riscv/include/asm/vdso/gettimeofday.h

csu/libc-start.c
LIB_START_MAIN

hexdump -x /proc/self/auxv
cat /proc/self/maps

process stack:
auxvec
env
arg
stack

glibc: 
dynamic linker
sysdeps/unix/sysv/linux/dl-sysdep.c
    `_dl_sysdep_start`
    `_dl_sysdep_parse_arguments`
sysdeps/unix/sysv/linux/dl-parse_auxv.h
    `_dl_parse_auxv`

libc init
sysdeps/unix/sysv/linux/dl-vdso-setup.h
    setup_vdso_pointers
elf/setup-vdso.h
    setup_vdso
elf/rtld.c
    dl_main

libc call
sysdeps/unix/sysv/linux/gettimeofday.c
    INLINE_VSYSCALL
sysdeps/unix/sysv/linux/sysdep-vdso.h
    INTERNAL_VSYSCALL_CALL / dl_vdso_gettimeofday

## kernel update vvar

kernel/time/timekeeping.c
timekeeping_update
kernel/time/vsyscall.c
update_vsyscall


kernel/time/time.c
SYSCALL settimeofday
do_sys_settimeofday64
kernel/time/vsyscall.c
update_vsyscall_tz

### shared object

相关代码以及如何生成。

### memory layout

cat /proc/self/maps

### so 加载

### gettimeofday 执行过程

### 最近的代码提交

### 可能的 patch

完善 RISC-V 上的 vDSO 支持的函数，现在只有 gettimeofday

[1]: https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/3711/original/LPC_vDSO.pdf
[2]: https://mirrors.edge.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.53
[3]: https://en.wikipedia.org/wiki/Address_space_layout_randomization
[4]: https://man7.org/linux/man-pages/man7/vdso.7.html
[5]: https://lwn.net/Articles/519085/

参考资料
- [什麼是 Linux vDSO 與 vsyscall？——發展過程](https://alittleresearcher.blogspot.com/2017/04/linux-vdso-and-vsyscall-history.html)