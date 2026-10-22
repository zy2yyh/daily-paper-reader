---
title: Device-Assisted Live Migration of RDMA Devices
title_zh: 设备辅助的RDMA设备实时迁移
authors: Liran Liss
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764795"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 面向高性能通信的RDMA设备实时迁移
tldr: 将HPC与AI等高性能工作负载迁移到云需要直接访问RDMA设备，但设备直通破坏了虚拟机与硬件的解耦，使实时迁移极为困难。本文提出设备辅助的迁移方案，设计通用设备-虚拟机监控器接口，并为NVIDIA ConnectX系列网卡实现迁移支持。该方案对虚拟机和网络对端均透明，能够在迁移中保持直接交互硬件设备间的连接，为云上RDMA通信的可迁移性提供了新思路。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 云上高性能工作负载需直接访问RDMA设备，但设备直通使虚拟机实时迁移难以实现。
method: 论文提出设备辅助迁移方案，设计通用设备-虚拟机监控器接口并为ConnectX网卡实现支持。
result: 该方案对虚拟机和网络对端透明，可在迁移中保持相互交互硬件设备之间的连接。
conclusion: 该工作为云环境中RDMA设备的高性能通信与可迁移性提供了可行方法。
---

## Abstract
Recently, we have seen growing pressure to move highperformance workloads, such as HPC and AI, to cloud environments that offer more affordable and manageable infrastructure. These workloads require direct access to RDMA devices for high-performance communication. Device passthrough, however, violates the decoupling between the guest OS and the underlying hardware, making Live Migration (LM) extremely challenging [29, 38, 40, 42, 48]. This paper presents a method for migrating a collection of directly interacting hardware devices in a manner that is transparent to both the VM and its network peers. We propose a device-assisted solution that includes (a) a generic device-hypervisor interface, (b) the design and implementation of LM support for the NVIDIA ConnectX family of network adapters, and (c) a novel scheme to quiesce direct communication over the memory fabric (e.g., PCIe). We demonstrate transparent migration of HPC and AI workloads in accelerated virtual environments. Our approach incurs no runtime overhead or performance degradation after migration and achieves sub-second downtimes even for VMs with high RDMA resource utilization.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向高性能通信的RDMA设备实时迁移。

### 2. 核心内容
将HPC与AI等高性能工作负载迁移到云需要直接访问RDMA设备，但设备直通破坏了虚拟机与硬件的解耦，使实时迁移极为困难。本文提出设备辅助的迁移方案，设计通用设备-虚拟机监控器接口，并为NVIDIA ConnectX系列网卡实现迁移支持。该方案对虚拟机和网络对端均透明，能够在迁移中保持直接交互硬件设备间的连接，为云上RDMA通信的可迁移性提供了新思路。

### 3. 对应检索需求
remote direct memory access transport for low-latency data center communication。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764795](https://dl.acm.org/doi/10.1145/3731569.3764795)
