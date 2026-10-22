---
title: "SWARM: Replicating Shared Disaggregated-Memory Data in No Time"
title_zh: SWARM：瞬时复制共享的解耦内存数据
authors: "Antoine Murat, Clément Burgelin, Athanasios Xygkis, Igor Zablotchi, Marcos K. Aguilera, Rachid Guerraoui"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3694715.3695945"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向解耦内存数据中心架构的单次往返复制协议
tldr: 内存解耦是提升数据中心资源利用率的新兴架构，但在解耦内存中复制共享数据难以兼顾低延迟与强一致性。本文提出SWARM复制方案，包含无等待的Safe-Guess复制协议与In-n-Out原子更新技术，实现常见情况下的单次往返读写、线性一致性与无等待活性。该工作为解耦内存数据中心提供了高效可靠的复制基础，与低延迟远程内存通信机制有一定方法关联。
source: SOSP-2024-ACM
selection_source: conference_retrieval
motivation: 解耦内存数据中心需要复制共享数据以保证容错，但现有方案难以兼顾低延迟与强一致性。
method: 提出SWARM复制方案，含Safe-Guess无等待协议与In-n-Out原子更新技术。
result: 实现常见情况下单次往返读写，同时保证线性一致性与无等待活性。
conclusion: 为解耦内存系统提供高效复制基础，与远程内存通信机制相关。
---

## Abstract
Memory disaggregation is an emerging data center architecture that improves resource utilization and scalability. Replication is key to ensure the fault tolerance of applications, but replicating shared data in disaggregated memory is hard. We propose SWARM (Swift WAit-free Replication in disaggregated Memory), the first replication scheme for in-disaggregated-memory shared objects to provide (1) single-roundtrip reads and writes in the common case, (2) strong consistency (linearizability), and (3) strong liveness (wait-freedom). SWARM makes two independent contributions. The first is Safe-Guess, a novel wait-free replication protocol with single-roundtrip operations. The second is In-n-Out, a novel technique to provide conditional atomic update and atomic retrieval of large buffers in disaggregated memory in one roundtrip. Using SWARM, we build SWARM-KV, a low-latency, strongly consistent and highly available disaggregated key-value store. We evaluate SWARM-KV and find that it has marginal latency overhead compared to an unreplicated key-value store, and that it offers much lower latency and better availability than FUSEE, a state-of-the-art replicated disaggregated key-value store.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向解耦内存数据中心架构的单次往返复制协议。

### 2. 核心内容
内存解耦是提升数据中心资源利用率的新兴架构，但在解耦内存中复制共享数据难以兼顾低延迟与强一致性。本文提出SWARM复制方案，包含无等待的Safe-Guess复制协议与In-n-Out原子更新技术，实现常见情况下的单次往返读写、线性一致性与无等待活性。该工作为解耦内存数据中心提供了高效可靠的复制基础，与低延迟远程内存通信机制有一定方法关联。

### 3. 对应检索需求
data center network architectures and communication mechanisms for large-scale clusters。

### 4. 来源与原文
- Source：SOSP-2024-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3694715.3695945](https://dl.acm.org/doi/10.1145/3694715.3695945)
