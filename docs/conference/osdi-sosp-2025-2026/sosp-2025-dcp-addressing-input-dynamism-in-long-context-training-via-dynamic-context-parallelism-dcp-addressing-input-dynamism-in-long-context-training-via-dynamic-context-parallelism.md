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
evidence: 降低长上下文分布式训练中不必要的通信开销
tldr: 现有上下文并行方法采用静态并行配置，忽视训练数据中序列长度与注意力模式的动态变化，导致多余通信开销与计算不均衡。DCP提出细粒度分块划分数据与计算，使块能灵活映射到设备上。该框架减少了不必要的通信并缓解负载失衡，提升了长上下文训练效率。其贡献在于为动态负载下的分布式训练通信优化提供了自适应并行方法。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 现有上下文并行采用静态配置，忽视序列长度与注意力模式的动态变化，造成多余通信与失衡。
method: 提出动态上下文并行框架，对数据与计算做细粒度分块划分，并灵活映射到设备。
result: 减少不必要的通信开销并缓解计算不均衡，提升长上下文训练效率。
conclusion: 该工作为动态负载下长上下文训练的通信优化提供了自适应并行方案。
---

## Abstract
Context parallelism has emerged as a key technique to support long-context training, a growing trend in generative AI for modern large models. However, existing context parallel methods rely on static parallelization configurations that overlook the dynamic nature of training data, specifically, the variability in sequence lengths and token relationships (i.e., attention patterns) across samples. As a result, these methods often suffer from unnecessary communication overhead and imbalanced computation. In this paper, we present DCP, a dynamic context parallel training framework that introduces fine-grained blockwise partitioning of both data and computation. By enabling flexible mapping of data and computation blocks to devices, DCP can adapt to varying sequence characteristics, effectively reducing communication and improving memory and computation balance. Micro-benchmarks demonstrate that DCP accelerates attention by 1.19x~2.45x under causal masks and 2.15x~3.77x under sparse attention patterns. Additionally, we observe up to 0.94x~1.16x end-to-end training speed-up for causal masks, and 1.00x~1.46x for sparse masks.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
降低长上下文分布式训练中不必要的通信开销。

### 2. 核心内容
现有上下文并行方法采用静态并行配置，忽视训练数据中序列长度与注意力模式的动态变化，导致多余通信开销与计算不均衡。DCP提出细粒度分块划分数据与计算，使块能灵活映射到设备上。该框架减少了不必要的通信并缓解负载失衡，提升了长上下文训练效率。其贡献在于为动态负载下的分布式训练通信优化提供了自适应并行方法。

### 3. 对应检索需求
How to optimize communication overhead in distributed training over data center fabrics。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764849](https://dl.acm.org/doi/10.1145/3731569.3764849)
