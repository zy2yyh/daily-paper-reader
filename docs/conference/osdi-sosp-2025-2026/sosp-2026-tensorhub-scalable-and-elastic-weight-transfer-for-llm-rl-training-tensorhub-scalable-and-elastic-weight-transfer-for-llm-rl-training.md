---
title: "TensorHub: Scalable and Elastic Weight Transfer for LLM RL Training"
title_zh: TensorHub：面向大模型强化学习训练的可扩展弹性权重传输
authors: "Chenhao Ye, Wencong Xiao, Remzi Arpaci-Dusseau"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3830418.3843859"
tags: ["query:dcn-comm"]
score: 7.0
evidence: 面向数千 GPU 的 TB 级权重传输与低数据搬移开销
tldr: 针对大模型强化学习训练中 TB 级权重需在数千 GPU 间频繁迁移、且集群动态伸缩时协调与存储开销高的问题，TensorHub 提出面向引用的存储抽象。该方法利用高度复制的模型权重，在不物理保存副本的情况下按需提供权重版本。系统降低了数据搬移、协调和存储开销，使弹性异构集群上的权重传输更可扩展，为分布式训练通信优化提供了新机制。
source: SOSP-2026-ACM
selection_source: conference_retrieval
motivation: 大模型强化学习需在数千 GPU 间传输 TB 级权重，集群动态伸缩时协调、数据搬移与存储开销很高。
method: TensorHub 提出面向引用的存储抽象，利用高度复制的权重按需提供版本，而不物理保存额外副本。
result: 该设计在保持低协调、低数据搬移和低存储开销的同时，支持异构集群上的弹性权重传输。
conclusion: 结果表明利用权重复制特性可显著提升大规模强化学习权重传输的可扩展性与弹性。
---

## Abstract
Modern LLM reinforcement learning (RL) workloads require a high-performance weight transfer system to scale training across heterogeneous compute resources. However, efficiently transferring terabyte-scale model weights across thousands of GPUs remains challenging because the system must accommodate clusters that dynamically scale up and down while keeping coordination, data movement, and storage overhead low. We introduce Reference-Oriented Storage (ROS), a new storage abstraction for RL weight transfer that exploits highly replicated model weights in place. ROS presents the illusion that certain versions of the model weights are stored and can be fetched on demand. Underneath, ROS does not physically store any copies of the weights; instead, it tracks the workers that hold these weights on GPUs for inference. Upon request, ROS directly uses them to serve reads. We build TensorHub, a production-quality system that instantiates the ROS idea with topology-aware transfer, model-parallel consistency, and fault tolerance. Evaluation shows that TensorHub saturates RDMA bandwidth and adapts to three distinct rollout workloads with minimal engineering effort. Specifically, TensorHub reduces total GPU stall time by up to 6.7× for standalone rollouts, accelerates weight updates for elastic rollouts by up to 4.8×, and cuts cross-datacenter rollout stall time by up to 19×. TensorHub has been deployed in ByteDance production to support cutting-edge RL training.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向数千 GPU 的 TB 级权重传输与低数据搬移开销。

### 2. 核心内容
针对大模型强化学习训练中 TB 级权重需在数千 GPU 间频繁迁移、且集群动态伸缩时协调与存储开销高的问题，TensorHub 提出面向引用的存储抽象。该方法利用高度复制的模型权重，在不物理保存副本的情况下按需提供权重版本。系统降低了数据搬移、协调和存储开销，使弹性异构集群上的权重传输更可扩展，为分布式训练通信优化提供了新机制。

### 3. 对应检索需求
How to optimize communication overhead in distributed training over data center fabrics。

### 4. 来源与原文
- Source：SOSP-2026-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3830418.3843859](https://dl.acm.org/doi/10.1145/3830418.3843859)
