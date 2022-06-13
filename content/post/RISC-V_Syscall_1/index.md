---
title: RISC-V Syscall 系列1：什么是 Syscall ?
subtitle: Welcome 👋 We know that first impressions are important, so we've populated your new site with some initial content to help you get familiar with everything in no time.

# Summary for listings and search engines
summary: 什么是 Syscall (系统调用)？ Syscall 该如何使用？

# Link this post with a project
projects: []

# Date published
date: '2022-06-13T00:00:00Z'

# Date updated
lastmod: '2022-06-14T00:00:00Z'

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

## 概览

![Linux_API](https://upload.wikimedia.org/wikipedia/commons/4/43/Linux_API.svg)

Syscall 又称为系统调用，它是操作系统内核给用户态程序提供的一组API，可以用来访问系统资源和内核提供的服务。比如用户态程序申请内存、读写文件等都需要通过 Syscall 完成。下面我们通过一段汇编代码来看看 Syscall 是如何使用的。

```asm
.data

msg:
    .ascii "Hello, world!\n"

.text
    .global _start

_start:
    li a7, 64
    li a0, 1
    la a1, msg
    la a2, 14
    ecall

    li a7, 93
    li a0, 0
    ecall
```

上面的代码通过系统调用往标准输出上打印 "Hello, world!".

![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2FLife-cc%2FdITXfLkXGx.png?alt=media&token=31158480-7224-4d2f-9348-fa8677b3570e)

RISC-V 中 通过 ecall 指令进行 Syscall 的调用。该指令会将CPU从用户态转换到内核态，并跳转到系统调用的入口处（关于 ecall 指令详细情况会在后面的系列中进行介绍）。

其中 a7 寄存器存储系统调用号（表示本次调用哪个 Syscall ）。write 的系统调用号为 64。a0 - a5 这6个寄存器分别用来表示第1个-第6个系统调用的参数。

系统调用号列表在 Linux 源码位置：include/uapi/asm-generic/unistd.h

```c
#define __NR_write 64c
__SYSCALL(__NR_write, sys_write)
  
#define __NR_exit 93
__SYSCALL(__NR_exit, sys_exit)
```

系统调用函数声明源码位置：include/linux/syscalls.h

```c
asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count);

asmlinkage long sys_exit(int error_code);
```

C 标准库提供了对 Syscall 的封装。

```c
#include <stdio.h>

int main() {
  printf("Hello, world!\n");
  return 0;
}
```

参考资料
- [syscall(2) — Linux manual page](https://man7.org/linux/man-pages/man2/syscall.2.html)
- [Linux kernel interfaces](https://en.wikipedia.org/wiki/Linux_kernel_interfaces)
