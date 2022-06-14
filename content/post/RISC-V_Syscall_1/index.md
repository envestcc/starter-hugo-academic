---
title: RISC-V Syscall 系列1：什么是 Syscall ?
subtitle: 

# Summary for listings and search engines
summary: 什么是 Syscall (系统调用)？ Syscall 该如何使用？

# Link this post with a project
projects: []

# Date published
date: '2022-06-13T00:00:00Z'

# Date updated
lastmod: '2022-06-14T11:30:00Z'

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
  - 教程
---

## 什么是 Syscall ?

![Linux_API](https://upload.wikimedia.org/wikipedia/commons/4/43/Linux_API.svg)

Syscall 又称为系统调用，它是操作系统内核给用户态程序提供的一组API，可以用来访问系统资源和内核提供的服务。比如用户态程序申请内存、读写文件等都需要通过 Syscall 完成。

通过 Linux 源码里可以看到(include/linux/syscalls.h)，大约有400多个 Syscall。其中一部分是兼容POSIX标准，另一些是 Linux 特有的。


## 如何调用 Syscall ?

应用程序想要调用 Syscall 有两种方式，分别是直接调用和使用 C 标准库。

### 直接调用

下面我们通过一段汇编代码来看看如何直接调用 Syscall.

```asm
.data

msg:
    .ascii "Hello, world!\n"

.text
    .global _start

_start:
    li a7, 64    # linux write syscall
    li a0, 1     # stdout
    la a1, msg   # address of string
    la a2, 14    # length of string
    ecall        # call linux syscall

    li a7, 93    # linux exit syscall
    li a0, 0     # return value
    ecall        # call linux syscall
```

上面的代码的功能是通过系统调用往标准输出上打印一串字符。

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FdITXfLkXGx.png?alt=media&token=31158480-7224-4d2f-9348-fa8677b3570e)

RISC-V 中通过`ecall`指令进行 Syscall 的调用。该指令会将CPU从用户态转换到内核态，并跳转到 Syscall 的入口处。通过 a7 寄存器来标识是哪个 Syscall. 至于调用 Syscall 要传递的参数则可以依次使用a0-a5这6个寄存器来存储。

> ecall 指令之前叫 scall，包括现在 Linux 源码里都用的是 scall，后来改为了 ecall。原因是该指令不止可以用来进行系统调用，可以提供更通用化的功能。

write 的系统调用号为64，所以上述代码里将64存储到a7中。该系统调用的参数有3个，第一个是文件描述符，第二个是要打印的字符串地址，第三个是字符串的长度，上述代码中将这三个参数分别存入到a0、a1、a2这三个寄存器中。

系统调用号列表可以在Linux源码中进行查看：include/uapi/asm-generic/unistd.h。

```c
#define __NR_write 64
  
#define __NR_exit 93
```

系统调用函数声明源码位置：include/linux/syscalls.h

```c
asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count);

asmlinkage long sys_exit(int error_code);
```

### C 标准库

直接使用汇编调用 Syscall 比较繁琐也不安全，[C标准库](https://en.wikipedia.org/wiki/C_standard_library)提供了对 Syscall 的封装。

![GNU C Library](https://en.wikipedia.org/wiki/Linux_kernel_interfaces#/media/File:Linux_kernel_System_Call_Interface_and_glibc.svg)

下面用一段C代码例子看看如何使用 Syscall ，这种方式大家都比较熟悉。

```c
#include <unistd.h>

int main() {
  write(1, "Hello, world!\n", 13);
  return 0;
}

```

使用下面的命令进行测试即可输出结果。
```
❯ cat testc.c
#include <unistd.h>

int main() {

  write(1, "Hello, world!\n", 14);
  return 0;
}
❯ 
❯ riscv64-linux-gnu-gcc -static testc.c -o testc
❯ qemu-riscv64 testc
Hello, world!

```

## 总结

本篇文章主要从 Syscall 使用者的角度，阐述了什么是 Syscall。然后以实际代码为例，展示了在 RISC-V 架构下应用程序如何使用汇编代码和C标准库两种方式调用 Syscall 。

系列文章预告：RISC-V Syscall 系列2： Syscall 过程分析

参考资料
- [System call](https://en.wikipedia.org/wiki/System_call)
- [syscall(2) — Linux manual page](https://man7.org/linux/man-pages/man2/syscall.2.html)
- [Linux kernel interfaces](https://en.wikipedia.org/wiki/Linux_kernel_interfaces)
- [RISC-V Assembly Programmer's Manual](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md)
- [RISC-V架构下利用QEMU进行GDB调试](https://zhuanlan.zhihu.com/p/517497012)
- [Risc-V Assembly Language Hello World](https://smist08.wordpress.com/2019/09/07/risc-v-assembly-language-hello-world/)
- [System Interface & Headers Reference](https://pubs.opengroup.org/onlinepubs/007908799/xshix.html)

