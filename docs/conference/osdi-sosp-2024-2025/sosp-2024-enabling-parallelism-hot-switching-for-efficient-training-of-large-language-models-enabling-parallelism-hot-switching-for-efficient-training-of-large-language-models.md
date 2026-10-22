---
title: Enabling Parallelism Hot Switching for Efficient Training of Large Language Models
title_zh: 面向大语言模型高效训练的并行策略热切换
authors: "Hao Ge, Fangcheng Fu, Haoyang Li, Xuanyu Wang, Sheng Lin, Yujie Wang, Xiaonan Nie, Hailin Zhang, Xupeng Miao, Bin Cui"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3694715.3695969"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 依据通信代价选择并行策略以优化大模型训练
tldr: 大模型训练通常假设样本负载一致而采用静态并行策略，但变长序列输入使该假设失效，导致性能次优。本文揭示静态策略带来的资源利用不足问题，提出在训练过程中动态热切换并行策略，依据内存、计算与通信代价自适应调整。实验表明该方法能有效跟踪负载变化、提升训练效率，为变负载场景下的分布式训练调度提供了新思路。
source: SOSP-2024-ACM
selection_source: conference_retrieval
motivation: 现有大模型训练系统采用静态并行策略，难以应对变长序列带来的负载波动，造成性能次优。
method: 论文揭示静态并行策略的利用不足问题，提出在训练过程中根据计算与通信代价动态热切换并行策略。
result: 实验显示动态切换并行策略可跟踪样本负载变化，相比静态策略获得更优训练性能。
conclusion: 该工作说明自适应并行策略对变负载大模型训练具有重要价值。
---

## Abstract
Training of large-scale deep learning models necessitates parallelizing the model and data across numerous devices, and the choice of parallelism strategy substantially depends on the training workloads such as memory consumption, computation cost, and communication cost. Current approaches generally assume uniform training workloads across samples in a given task. Thus, existing systems are designed to adopt a static parallelism strategy throughout one training process. Nevertheless, when training models with sequence inputs, this assumption fails due to the sequence length variation across samples. Consequently, training with a static parallelism strategy would result in sub-optimal performance. In this paper, we first reveal the under-explored fact that the optimal parallelism strategy varies even for the sequences within a single mini-batch. Motivated by this, we present HotSPa, a novel system that adopts multiple parallelism strategies for efficient training with sequence inputs. To be specific, given a mini-batch of training sequences, HotSPa partitions them into multiple groups and applies different parallelism strategies to process each group individually. To enable the hot switching between strategies, HotSPa transfers model parameters and accumulated gradients among the devices on the fly. Significant solutions are proposed with the hope of seamless and rapid parallelism hot switching. Firstly, we design a graph compiler, which generates distributed computation graphs for different parallelism strategies simultaneously, and orchestrates them to share a single model storage backbone. Secondly, we develop a simple yet effective hot switch planner, which heuristically deduces communication plans to accelerate the transition of model partitioning given any pairs of strategies. Extensive experiments on large language model training demonstrate that HotSPa can be up to 2.99× faster than Megatron-LM and DeepSpeed that utilize static parallelism strategies. Source code is available: https://github.com/PKU-DAIR/Hetu.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
依据通信代价选择并行策略以优化大模型训练。

### 2. 核心内容
大模型训练通常假设样本负载一致而采用静态并行策略，但变长序列输入使该假设失效，导致性能次优。本文揭示静态策略带来的资源利用不足问题，提出在训练过程中动态热切换并行策略，依据内存、计算与通信代价自适应调整。实验表明该方法能有效跟踪负载变化、提升训练效率，为变负载场景下的分布式训练调度提供了新思路。

### 3. 对应检索需求
communication optimization for distributed training workloads in data center networks。

### 4. 来源与原文
- Source：SOSP-2024-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3694715.3695969](https://dl.acm.org/doi/10.1145/3694715.3695969)
