---
title: RISC-V eBPF 系列 2：bpf SYSCALL
subtitle: 

# Summary for listings and search engines
summary: 介绍 bpf 系统调用的功能及其源码实现

# Link this post with a project
projects: []

# Date published
date: '2022-11-29T00:00:00Z'

# Date updated
lastmod: '2022-11-29T11:30:00Z'

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
  - eBPF
  - RISC-V

categories:
  - 技术
---


## 前言

上一篇文章中介绍了 eBPF 技术的背景，本文主要围绕 bpf SYSCALL 进行介绍，包括它的使用以及具体的源码实现。

## bpf SYSCALL

bpf SYSCALL 是操作系统内核给用户程序提供的用于 eBPF 的编程接口。用户空间所有的 eBPF 相关函数，归根结底都是对于 bpf SYSCALL 的包装。所以直接学习它有助于我们更好理解其使用和原理。

总体来说，bpf SYSCALL 的功能主要包含两部分：
* 对 eBPF maps 数据的增删改查
* 对 eBPF 程序进行验证和加载

### bpf 函数原型

```c
#include <linux/bpf.h>

int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

bpf 函数包含 3 个参数：
* cmd：本次系统调用要执行的动作，包括对 map 的增删改查以及程序的加载
* attr：一个大的联合结构体，包含了执行各种操作所需要的数据
* size：attr 结构的大小

### eBPF maps

map 是一种可以用来存储不同类型数据的通用数据结构，可以使用它在不同 eBPF 程序之间以及和用户程序之间共享数据。

#### types

Hash-table map
* 查询性能好
* 更新是原子操作

Array map
* 最快的查询性能
* 内存是预分配的
* key 必须是 4 字节
* 无法删除元素
* 更新操作不是原子操作

Program array map
* value 是指向另一个 eBPF program 的文件描述符
* key 和 value 都是 4 字节

#### map create

```c
int
bpf_create_map(enum bpf_map_type map_type,
                unsigned int key_size,
                unsigned int value_size,
                unsigned int max_entries)
{
    union bpf_attr attr = {
        .map_type    = map_type,
        .key_size    = key_size,
        .value_size  = value_size,
        .max_entries = max_entries
    };

    return bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
}
```
创建 map 时，cmd 参数设置为 BPF_MAP_CREATE。然后需要指定 map 类型（map_type），键的大小（key_size），值的大小（value_size）以及元素容量（max_entries），这些统一放入 attr 参数里即可。最后返回一个文件描述符。

#### map lookup

```c
int
bpf_lookup_elem(int fd, const void *key, void *value)
{
    union bpf_attr attr = {
        .map_fd = fd,
        .key    = ptr_to_u64(key),
        .value  = ptr_to_u64(value),
    };

    return bpf(BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr));
}
```
需要在 map 中查找元素时，cmd 参数设置为 BPF_MAP_LOOKUP_ELEM。然后指定 map 对应的文件描述符，键的地址，查询值保存的地址，一起放入 attr 参数里即可。

#### map update

```c
int
bpf_update_elem(int fd, const void *key, const void *value,
                uint64_t flags)
{
    union bpf_attr attr = {
        .map_fd = fd,
        .key    = ptr_to_u64(key),
        .value  = ptr_to_u64(value),
        .flags  = flags,
    };

    return bpf(BPF_MAP_UPDATE_ELEM, &attr, sizeof(attr));
}
```
需要更新 map 中元素时，cmd 参数设置为 BPF_MAP_UPDATE_ELEM。然后指定 map 对应的文件描述符，键的地址，值的地址，以及更新标记一起放入 attr 参数里即可。

更新标记（flags）有以下三种选项：
* BPF_ANY：创建新元素或者更新已存在的
* BPF_NOEXIST：只在不存在时创建新元素
* BPF_EXIST：只在存在时更新元素

#### map delete

```c
int
bpf_delete_elem(int fd, const void *key)
{
    union bpf_attr attr = {
        .map_fd = fd,
        .key    = ptr_to_u64(key),
    };

    return bpf(BPF_MAP_DELETE_ELEM, &attr, sizeof(attr));
}
```
需要删除 map 中元素时，cmd 参数设置为 BPF_MAP_DELETE_ELEM。然后指定 map 对应的文件描述符和键的地址，一起放入 attr 参数里即可。


### eBPF programs

eBPF programs 的相关操作包括：加载（load），附加（attach），链接（link）以及固定（pin）。

#### load

通过加载（load）过程，将指令注入内核。程序通过验证器会进行许多检查并可能重写一些指令（特别是对于 map 的访问）。如果启用了 JIT 编译，则程序可能是 JIT 进行重新编译的。

```c
char bpf_log_buf[LOG_BUF_SIZE];

int
bpf_prog_load(enum bpf_prog_type type,
              const struct bpf_insn *insns, int insn_cnt,
              const char *license)
{
    union bpf_attr attr = {
        .prog_type = type,
        .insns     = ptr_to_u64(insns),
        .insn_cnt  = insn_cnt,
        .license   = ptr_to_u64(license),
        .log_buf   = ptr_to_u64(bpf_log_buf),
        .log_size  = LOG_BUF_SIZE,
        .log_level = 1,
    };

    return bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
}
```

需要载入一段 eBPF 程序到内核时，cmd 参数设置为 BPF_PROG_LOAD。attr 中需要指定的参数包括：
* eBPF 程序类型（prog_type）
* 程序指令数组（insns）
* 指令数量（insn_cnt）
* 授权许可证（license）
* 日志缓冲区（log_buf）
* 日志缓冲区大小（log_size）
* 日志级别（log_level）

#### program types

目前支持的 program types 包括：
* BPF_PROG_TYPE_SOCKET_FILTER
* BPF_PROG_TYPE_KPROBE
* BPF_PROG_TYPE_SCHED_CLS
* BPF_PROG_TYPE_SCHED_ACT
* BPF_PROG_TYPE_TRACEPOINT
* BPF_PROG_TYPE_XDP
* BPF_PROG_TYPE_PERF_EVENT
* BPF_PROG_TYPE_CGROUP_SKB
* BPF_PROG_TYPE_CGROUP_SOCK
* BPF_PROG_TYPE_LWT_IN
* BPF_PROG_TYPE_LWT_OUT
* BPF_PROG_TYPE_LWT_XMIT
* BPF_PROG_TYPE_SOCK_OPS
* BPF_PROG_TYPE_SK_SKB
* BPF_PROG_TYPE_CGROUP_DEVICE
* BPF_PROG_TYPE_SK_MSG
* BPF_PROG_TYPE_RAW_TRACEPOINT
* BPF_PROG_TYPE_CGROUP_SOCK_ADDR
* BPF_PROG_TYPE_LWT_SEG6LOCAL
* BPF_PROG_TYPE_LIRC_MODE2
* BPF_PROG_TYPE_SK_REUSEPORT
* BPF_PROG_TYPE_FLOW_DISSECTOR
* BPF_PROG_TYPE_CGROUP_SYSCTL
* BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE
* BPF_PROG_TYPE_CGROUP_SOCKOPT

不同的 eBPF program type 主要区别在于：
* 能使用的内核帮助函数（kernel helper functions）集合
* 程序传入的上下文（context）格式

#### attach

eBPF programs 载入内核后可以附加（attach）到不同的事件（event）进行触发，包括网络包（network packets），追踪事件（tracing events）等。

attach 具体的方法取决于 program 或者 event 的类型。例如可以使用 setsockopt 将 program 附加到网络包事件上。
```c
setsockopt(sockfd, SOL_SOCKET, SO_ATTACH_BPF, &prog_fd, sizeof(prog_fd));
```

而对于 cgroup 相关的类型可以使用 `bpf(BPF_PROG_ATTACH,...)` 系统调用进行。

如果所有类型都由 bpf 系统调用统一进行 attach 会比较一致。但 `BPF_PROG_ATTACH` 是内核新增的特性，因为历史遗留原因，导致了不同类型的 attach 方法不一致，希望在后续内核版本中可以进行统一。

#### link

当加载 BPF 的程序关闭时，由于 BPF 程序引用归零，就会被内核卸载。那么如何在程序关闭时保持 BPF 程序的运行呢？此时可以就使用链接（link）。

BPF 程序可以 attach 到一个链接，而不是传统的钩子。链接本身 attach 到内核的钩子。这样有两个好处：
* 可以固定这种链接，在加载程序退出时保持 BPF 继续运行。
* 更容易跟踪程序中持有的引用，以确保加载程序意外退出时没有 BPF 程序被加载。

可以使用 `bpf(BPF_LINK_CREATE,...)` 系统调用来创建链接。

#### pin

pin 是一种保持 BPF 对象 (程序，映射，链接) 引用的方法。通过 `bpf(BPF_OBJ_PIN, ...)` 这个系统调用完成，这会在 eBPF 的虚拟文件系统中创建一个路径，并且之后可以用 `open()` 该路径来获取该 BPF 对象的文件描述符。只要一个对象被固定住，他就会一直保留在内核中，不需要 pin 或者 map 就可运行它。只要存在其他引用 (文件描述符。附加到一些钩子，或者被其他程序引用) 程序就会一直加载在内核中，attach 之后可以直接运行。

## bpf 源码分析

bpf SYSCALL 在内核中对应的函数为 `__sys_bpf`：
```c
// kernel/bpf/syscall.c:4595
static int __sys_bpf(int cmd, bpfptr_t uattr, unsigned int size)
{
  ...
  err = bpf_check_uarg_tail_zero(uattr, sizeof(attr), size); // 检查 uattr 参数是否过大等
  ...
  if (copy_from_bpfptr(&attr, uattr, size) != 0) // attr 从用户态拷贝到内核态
  ...
  switch (cmd) { // 根据 cmd 类型调用对于的处理函数
    ...
    case BPF_PROG_LOAD:
		err = bpf_prog_load(&attr, uattr); // 加载 bpf 程序
		break;
    ...
  }
}
```

`__sys_bpf` 函数内主要包含两部分逻辑：
1. 权限和参数校验：包括 bpf 功能是否打开，attr 参数是否过大等
2. 针对不同的 cmd 调用对应的处理函数：目前 cmd 支持的类型比较多，后面主要介绍加载 eBPF 程序


### bpf_prog_load

当执行加载 eBPF 程序功能时会调用 `bpf_prog_load` 方法。下面介绍了该方法的主要内容。

```c
// kernel/bpf/syscall.c:2207
static int bpf_prog_load(union bpf_attr *attr, bpfptr_t uattr)
{
  ...
  // 开源许可证判断
  is_gpl = license_is_gpl_compatible(license); 
  // 限制 eBPF 程序指令数量
  if (attr->insn_cnt == 0 ||
	    attr->insn_cnt > (bpf_capable() ? BPF_COMPLEXITY_LIMIT_INSNS : BPF_MAXINSNS)) 
		return -E2BIG;
  // 如果是 net 相关类型，判断所需权限是否满足
  if (is_net_admin_prog_type(type) && !capable(CAP_NET_ADMIN) && !capable(CAP_SYS_ADMIN)) 
		return -EPERM;
  // 如果是追踪相关类型，判断权限是否满足
  if (is_perfmon_prog_type(type) && !perfmon_capable()) 
		return -EPERM;
  ...
  // 给 struct bpf_prog 申请内存，该结构是 bpf 在内核中的实例
  prog = bpf_prog_alloc(bpf_prog_size(attr->insn_cnt), GFP_USER); 
  ...
  // 拷贝 bpf 字节码到内核
  if (copy_from_bpfptr(prog->insns,
			     make_bpfptr(attr->insns, uattr.is_kernel),
			     bpf_prog_insn_size(prog)) != 0) 
  ...
  // bpf verify 机制核心
  err = bpf_check(&prog, attr, uattr); 
  // bpf jit 机制核心，将 bpf 字节码编译为目标平台汇编代码
  prog = bpf_prog_select_runtime(prog, &err); 
  ...
  // 打印一条 prog load 的 audit 信息
  bpf_audit_prog(prog, BPF_AUDIT_LOAD);
  //返回给应用层 bpf prog 的 fd 信息，后续应用层用该 fd 进行操作
  err = bpf_prog_new_fd(prog);
}
```

函数里的 `bpf_check` 和 `bpf_prog_select_runtime` 分别代表了 bpf 的 verify 和 JIT 两个核心机制，准备在后面的文章继续进行分析。

## 总结

本文主要介绍了 bpf 技术中涉及的 map 和 program 两个对象，并示例如何用 bpf SYSCALL 进行操作。之后还分析了加载 program 部分的源码，其中更细节的 verify 和 JIT 部分会在后续的文章中进行分析。

## 参考文章

* [bpf(2) — Linux manual page][1]
* [eBPF - difference between loading, attaching, and linking?][2]
* [BPF 之路三如何运行 BPF 程序][3]
* [eBPF 代码流程分析][4]
* [BPF 之路一 bpf 系统调用][5]

[1]: https://man7.org/linux/man-pages/man2/bpf.2.html
[2]: https://stackoverflow.com/questions/68278120/ebpf-difference-between-loading-attaching-and-linking
[3]: https://www.anquanke.com/post/id/265887
[4]: https://zhuanlan.zhihu.com/p/449814311
[5]: https://www.anquanke.com/post/id/263803