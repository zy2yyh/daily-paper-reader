---
title: "Oasis: Pooling PCIe Devices Over CXL to Boost Utilization"
title_zh: Oasis：通过CXL池化PCIe设备以提升利用率
authors: "Yuhong Zhong, Daniel S. Berger, Enrique Saurez, Dan R. K. Ports, Antonis Psistakis, Joshua Fried, Asaf Cidon"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764812"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 在数据中心通过CXL以软件方式池化并路由PCIe设备流量
tldr: 针对PCIe设备在云平台中常被低利用、而基于PCIe交换机的池化方案昂贵且不灵活的问题，Oasis利用已部署的CXL内存池作为主机与设备间的数据通路，以软件方式池化PCIe设备。系统提供控制面与数据面，在跨主机边界映射并路由设备流量。该工作提升了数据中心设备利用率，与数据中心网络通信机制有一定关联但非集合通信算法。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 云平台中PCIe设备常被低利用，基于PCIe交换机的池化方案昂贵且不灵活。
method: Oasis以已部署的CXL内存池为数据通路，提供控制面与数据面，软件化地映射并路由跨主机的PCIe设备流量。
result: 该方案提升了PCIe设备利用率并降低总拥有成本。
conclusion: 表明CXL内存池可作为高效数据通路支撑数据中心设备池化。
---

## Abstract
PCIe devices, such as NICs and SSDs, are frequently underutilized in cloud platforms. PCIe device pools, in which multiple hosts can share a set of PCIe devices, could increase PCIe device utilization and reduce their total cost of ownership. The main way to achieve PCIe device pools today is via PCIe switches, but they are expensive and inflexible. We design Oasis,1 a system that pools PCIe devices in software over CXL memory pools. CXL memory pools are already being deployed to boost datacenter memory utilization and reduce costs. Once CXL pools are in place, they can serve as an efficient data path between hosts and PCIe devices. Oasis provides a control plane and datapath over CXL pools, mapping and routing PCIe device traffic across host boundaries. PCIe devices with different functionalities can be supported by adding an Oasis engine for each device class. We implement an Oasis network engine to demonstrate NIC pooling. Our evaluation shows that Oasis improves the NIC utilization by 2× and handles NIC failover with only a 38 ms interruption.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
在数据中心通过CXL以软件方式池化并路由PCIe设备流量。

### 2. 核心内容
针对PCIe设备在云平台中常被低利用、而基于PCIe交换机的池化方案昂贵且不灵活的问题，Oasis利用已部署的CXL内存池作为主机与设备间的数据通路，以软件方式池化PCIe设备。系统提供控制面与数据面，在跨主机边界映射并路由设备流量。该工作提升了数据中心设备利用率，与数据中心网络通信机制有一定关联但非集合通信算法。

### 3. 对应检索需求
data center network architectures and communication mechanisms for large-scale clusters。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764812](https://dl.acm.org/doi/10.1145/3731569.3764812)
