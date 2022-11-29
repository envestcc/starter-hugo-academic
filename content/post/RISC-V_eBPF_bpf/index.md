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

#### bpf 函数原型

```c
#include <linux/bpf.h>

int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

bpf 函数包含 3 个参数：
* cmd：本次系统调用要执行的动作，包括对 map 的增删改查以及程序的加载
* attr：一个大的联合结构体，包含了执行各种操作所需要的数据
* size：attr 结构的大小

下面针对各种操作依次进行说明。

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

#### 


### eBPF programs



## bpf 源码分析

## 总结

## 参考文章

* [bpf(2) — Linux manual page][1]

[1]: https://man7.org/linux/man-pages/man2/bpf.2.html