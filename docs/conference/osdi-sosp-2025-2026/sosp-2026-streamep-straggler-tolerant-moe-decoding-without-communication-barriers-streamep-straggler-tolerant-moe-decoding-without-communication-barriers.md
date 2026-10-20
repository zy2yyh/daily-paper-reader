---
title: "StreamEP: Straggler-Tolerant MoE Decoding without Communication Barriers"
title_zh: StreamEP：无通信屏障的容错MoE解码
authors: "Shaoyu Wang, Jaeyong Song, Yanqi Zhou, Geon-Woo Kim, Seo Jin Park"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3830418.3843869"
tags: ["query:dcn-comm"]
score: 9.0
evidence: 消除MoE专家并行解码中的集合通信屏障
tldr: 该论文针对MoE专家并行解码依赖逐层集合通信、在共享且非均匀互连的商品GPU集群上被最慢传输拖慢的问题，提出无通信屏障的StreamEP范式。每个GPU独立接收令牌、执行相应层并尽早转发结果，从而避免全局同步。作者构建StreamInfer系统并在集群上评估，表明该方法能提升解码吞吐并容忍掉队者。该工作为数据中心内分布式训练与推理的集合通信优化提供了新思路。
source: SOSP-2026-ACM
selection_source: conference_retrieval
motivation: MoE专家并行依赖逐层集合通信，在商品GPU集群的非均匀互连上会被最慢传输串行化，形成通信屏障。
method: StreamEP让每个GPU独立接收令牌、执行对应专家层并尽早转发结果，从而彻底移除集合通信屏障。
result: 基于StreamInfer的实现表明，该范式可在共享互连上提升解码吞吐并容忍掉队者，避免全局同步开销。
conclusion: 研究表明去除集合通信屏障可改善MoE推理在商品集群上的可扩展性，为分布式推理通信优化提供新路径。
---

## Abstract
Mixture-of-Experts (MoE) models use expert parallelism (EP) to distribute experts across GPUs, relying on collective communication that synchronizes every device at every layer. To minimize communication latency, hyperscalers provision abundant network bandwidth in data centers and deploy MoE-optimized stacks that require Hopper-generation GPU features. But on commodity GPU clusters with shared and nonuniform interconnects, this communication barrier serializes execution behind the slowest transfer. We present stream expert parallelism (StreamEP), a paradigm that removes the collective communication barrier entirely: each GPU independently receives tokens, executes the appropriate layer, and forwards results as they become ready. We build StreamInfer, a serving system that realizes StreamEP on vanilla NCCL. StreamInfer introduces per-layer token queues for asynchronous token management, a defragmenting scheduler that consolidates partial batches while guaranteeing forward progress, and a two-phase communicator for asynchronous and variable-size GPU-to-GPU transfers. On 16×L40S and 16×A100 clusters with 200 Gbps inter-node networking, StreamInfer achieves up to 1.7× higher decoding throughput than traditional EP, demonstrating that efficient large-scale MoE serving is achievable on commodity hardware.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
消除MoE专家并行解码中的集合通信屏障。

### 2. 核心内容
该论文针对MoE专家并行解码依赖逐层集合通信、在共享且非均匀互连的商品GPU集群上被最慢传输拖慢的问题，提出无通信屏障的StreamEP范式。每个GPU独立接收令牌、执行相应层并尽早转发结果，从而避免全局同步。作者构建StreamInfer系统并在集群上评估，表明该方法能提升解码吞吐并容忍掉队者。该工作为数据中心内分布式训练与推理的集合通信优化提供了新思路。

### 3. 对应检索需求
communication optimization for distributed training workloads in data center networks。

### 4. 来源与原文
- Source：SOSP-2026-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3830418.3843869](https://dl.acm.org/doi/10.1145/3830418.3843869)
