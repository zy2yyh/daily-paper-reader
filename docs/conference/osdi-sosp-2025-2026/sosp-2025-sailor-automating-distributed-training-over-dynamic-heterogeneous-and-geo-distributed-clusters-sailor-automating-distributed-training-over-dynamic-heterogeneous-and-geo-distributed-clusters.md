---
title: "Sailor: Automating Distributed Training over Dynamic, Heterogeneous, and Geo-distributed Clusters"
title_zh: Sailor：在动态、异构与地理分布式集群上自动化分布式训练
authors: "George Manos, Ixeia Sánchez Périz, Qinghao Hu, Tiancheng Chen, Berk Buzcu, Song Han, Pamela Delgado, Ana Klimovic"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764839"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 面向异构跨区集群的分布式训练自动化，处理掉队者与作业配置搜索
tldr: 论文针对单一可用区难以凑齐大规模同构高端GPU、而异构与跨区资源又带来掉队者和庞大配置搜索空间的问题，提出Sailor系统，将高效的搜索空间探索算法与运行时间、显存开销预测相结合，自动为分布式训练选择作业配置与资源编排。在异构、地理分布且动态变化的集群上，Sailor 能自动完成配置搜索并获得更高吞吐与更优成本。该工作说明自动化配置搜索可让训练真正用上跨区异构算力。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 单一可用区难以凑齐大规模同构高端GPU，异构与跨区资源虽可降本增效，却带来掉队者与配置搜索难题。
method: 提出Sailor系统，结合高效搜索空间探索算法与运行时、显存开销预测模型，自动为异构跨区资源选择训练配置。
result: 通过自动配置搜索与资源编排，在异构、跨区且动态可用的资源上获得可行且高性价比的训练吞吐。
conclusion: 表明自动化配置搜索与资源编排能让分布式训练有效利用异构、跨区与动态可用算力。
---

## Abstract
The high GPU demand of ML training makes it hard to allocate large homogeneous clusters of high-end GPUs in a single availability zone. Leveraging heterogeneous GPUs available within and across zones can improve throughput at a reasonable cost. However, training ML models on heterogeneous resources introduces significant challenges, such as stragglers and a large search space of possible job configurations. Current systems lack support for efficiently training models on heterogeneous resources. We present Sailor, a system that automates distributed training over heterogeneous, geo-distributed, and dynamically available resources. Sailor combines an efficient search space exploration algorithm, accurate runtime and memory footprint simulation, and a distributed training framework that supports different types of heterogeneity to optimize training throughput and cost.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向异构跨区集群的分布式训练自动化，处理掉队者与作业配置搜索。

### 2. 核心内容
论文针对单一可用区难以凑齐大规模同构高端GPU、而异构与跨区资源又带来掉队者和庞大配置搜索空间的问题，提出Sailor系统，将高效的搜索空间探索算法与运行时间、显存开销预测相结合，自动为分布式训练选择作业配置与资源编排。在异构、地理分布且动态变化的集群上，Sailor 能自动完成配置搜索并获得更高吞吐与更优成本。该工作说明自动化配置搜索可让训练真正用上跨区异构算力。

### 3. 对应检索需求
communication optimization for distributed training workloads in data center networks。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764839](https://dl.acm.org/doi/10.1145/3731569.3764839)
