---
title: RISC-V eBPF 系列 1：eBPF 简介
subtitle: 

# Summary for listings and search engines
summary: 介绍 eBPF 基本功能和原理，并分析目前主要的应用场景和存在的问题，最后介绍内核最新的相关动态

# Link this post with a project
projects: []

# Date published
date: '2022-11-02T00:00:00Z'

# Date updated
lastmod: '2022-11-02T11:30:00Z'

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

> eBPF 是一项革命性技术，它能在内核中运行沙箱程序（sandbox programs），而无需修改内核源码或者加载内核模块。

## 背景

### BPF

BPF (Berkeley Packet Filter) 是类 Unix 系统上数据链路层的一种原始接口，提供原始链路层封包的收发。Linux 内核采用 BPF 作为网络数据包过滤技术。
包括像 tcpdump 的底层也是采用 BPF 作为底层包过滤技术。

[BPF Overview](BPF_overview.png)

### eBPF

eBPF (extended Berkeley Packet Filter) 是在 BPF 基础上经过重新设计，逐步演进为一个通用执行引擎，可用于开发性能分析工具等多种场景。

eBPF 最早出现在 3.18 内核版本中，此后原来的 BPF 就被称为经典 BPF，缩写 cBPF（classic BPF）。


## 基本功能

## 基本原理

## 应用场景

eBPF 相关的知名的开源项目包括但不限于以下：

Facebook 高性能 4 层负载均衡器 Katran；
Cilium 为下一代微服务 ServiceMesh 打造了具备 API 感知和安全高效的容器网络方案；底层主要使用 XDP 和 TC 等相关技术；
IO Visor 项目开源的 BCC、BPFTrace 和 Kubectl-Trace：BCC 提供了更高阶的抽象，可以让用户采用 Python、C++ 和 Lua 等高级语言快速开发 BPF 程序；BPFTrace 采用类似于 awk 语言快速编写 eBPF 程序；Kubectl-Trace 则提供了在 kubernetes 集群中使用 BPF 程序调试的方便操作；
CloudFlare 公司开源的 eBPF Exporter 和 bpf-tools：eBPF Exporter 将 eBPF 技术与监控 Prometheus 紧密结合起来；bpf-tools 可用于网络问题分析和排查；

## 存在的问题

## 最新动态

## 示例

[1]: https://ebpf.io/
[2]: https://www.tcpdump.org/papers/bpf-usenix93.pdf