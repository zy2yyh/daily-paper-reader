---
title: "DCP: Addressing Input Dynamism In Long-Context Training via Dynamic Context Parallelism"
title_zh: DCP：通过动态上下文并行应对长上下文训练中的输入动态性
authors: "Chenyu Jiang, Zhenkun Cai, Ye Tian, Yida Wang, Chuan Wu"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764849"
tags: ["query:dcn-comm"]
score: 7.0
evidence: 动态上下文并行降低分布式长上下文训练中的通信开销
tldr: 现有上下文并行方法采用静态切分配置，忽视样本序列长度与注意力模式的动态差异，导致多余的通信开销与计算不均衡。DCP提出动态上下文并行训练框架，对数据和计算做细粒度分块，并灵活地把块映射到设备上，从而按样本动态调整并行划分。实验表明该方案在长上下文训练中显著降低通信量并改善负载均衡，提升训练效率。其贡献在于把输入动态性纳入并行通信优化，对数据中心分布式训练通信优化有直接参考价值。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 静态上下文并行配置忽视样本序列长度与注意力模式的动态差异，造成通信浪费与负载不均。
method: 提出细粒度分块的数据与计算划分，并动态将块映射到设备以实现弹性并行。
result: 在长上下文训练中减少了不必要的通信开销并缓解了计算不均衡。
conclusion: 表明按输入动态调整并行策略可有效优化分布式训练的通信效率。
---

## Abstract
Context parallelism has emerged as a key technique to support long-context training, a growing trend in generative AI for modern large models. However, existing context parallel methods rely on static parallelization configurations that overlook the dynamic nature of training data, specifically, the variability in sequence lengths and token relationships (i.e., attention patterns) across samples. As a result, these methods often suffer from unnecessary communication overhead and imbalanced computation. In this paper, we present DCP, a dynamic context parallel training framework that introduces fine-grained blockwise partitioning of both data and computation. By enabling flexible mapping of data and computation blocks to devices, DCP can adapt to varying sequence characteristics, effectively reducing communication and improving memory and computation balance. Micro-benchmarks demonstrate that DCP accelerates attention by 1.19x~2.45x under causal masks and 2.15x~3.77x under sparse attention patterns. Additionally, we observe up to 0.94x~1.16x end-to-end training speed-up for causal masks, and 1.00x~1.46x for sparse masks.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
动态上下文并行降低分布式长上下文训练中的通信开销。

### 2. 核心内容
现有上下文并行方法采用静态切分配置，忽视样本序列长度与注意力模式的动态差异，导致多余的通信开销与计算不均衡。DCP提出动态上下文并行训练框架，对数据和计算做细粒度分块，并灵活地把块映射到设备上，从而按样本动态调整并行划分。实验表明该方案在长上下文训练中显著降低通信量并改善负载均衡，提升训练效率。其贡献在于把输入动态性纳入并行通信优化，对数据中心分布式训练通信优化有直接参考价值。

### 3. 对应检索需求
communication optimization for distributed training workloads in data center networks。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764849](https://dl.acm.org/doi/10.1145/3731569.3764849)
