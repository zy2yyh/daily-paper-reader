---
title: Device-Assisted Live Migration of RDMA Devices
title_zh: 设备辅助的RDMA设备热迁移
authors: Liran Liss
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764795"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向云中高性能通信的RDMA设备热迁移
tldr: 该论文针对云环境中HPC与AI工作负载依赖RDMA设备直通、却难以进行热迁移的问题，提出设备辅助的迁移方案。方法包括通用设备-虚拟机监控器接口，以及面向NVIDIA ConnectX网卡的迁移支持，使迁移对虚拟机和网络对端透明。该工作降低了RDMA设备直通与云基础设施解耦之间的矛盾，为高性能通信资源的弹性管理提供了可行路径。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: HPC与AI工作负载迁移到云需要RDMA直通，但设备直通破坏虚拟机与硬件解耦，使热迁移十分困难。
method: 论文提出设备辅助的RDMA设备热迁移方法，设计通用设备-虚拟机监控器接口，并为NVIDIA ConnectX网卡实现迁移支持。
result: 该方法使一组直接交互的硬件设备能够透明迁移，对虚拟机和网络对端均不暴露迁移过程。
conclusion: 研究表明设备辅助方案可缓解RDMA直通与云热迁移的矛盾，为高性能通信设备的弹性运维提供支持。
---

## Abstract
Recently, we have seen growing pressure to move highperformance workloads, such as HPC and AI, to cloud environments that offer more affordable and manageable infrastructure. These workloads require direct access to RDMA devices for high-performance communication. Device passthrough, however, violates the decoupling between the guest OS and the underlying hardware, making Live Migration (LM) extremely challenging [29, 38, 40, 42, 48]. This paper presents a method for migrating a collection of directly interacting hardware devices in a manner that is transparent to both the VM and its network peers. We propose a device-assisted solution that includes (a) a generic device-hypervisor interface, (b) the design and implementation of LM support for the NVIDIA ConnectX family of network adapters, and (c) a novel scheme to quiesce direct communication over the memory fabric (e.g., PCIe). We demonstrate transparent migration of HPC and AI workloads in accelerated virtual environments. Our approach incurs no runtime overhead or performance degradation after migration and achieves sub-second downtimes even for VMs with high RDMA resource utilization.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向云中高性能通信的RDMA设备热迁移。

### 2. 核心内容
该论文针对云环境中HPC与AI工作负载依赖RDMA设备直通、却难以进行热迁移的问题，提出设备辅助的迁移方案。方法包括通用设备-虚拟机监控器接口，以及面向NVIDIA ConnectX网卡的迁移支持，使迁移对虚拟机和网络对端透明。该工作降低了RDMA设备直通与云基础设施解耦之间的矛盾，为高性能通信资源的弹性管理提供了可行路径。

### 3. 对应检索需求
remote direct memory access transport for low-latency data center communication。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764795](https://dl.acm.org/doi/10.1145/3731569.3764795)
