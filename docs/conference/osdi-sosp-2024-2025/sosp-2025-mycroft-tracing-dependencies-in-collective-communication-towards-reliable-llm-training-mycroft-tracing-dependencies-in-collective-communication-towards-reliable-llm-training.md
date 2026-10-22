---
title: "Mycroft: Tracing Dependencies in Collective Communication Towards Reliable LLM Training"
title_zh: Mycroft：追踪集合通信中的依赖以实现可靠LLM训练
authors: "Yangtao Deng, Lei Zhang, Xiaoyun Zhi, Lei Wang, Gaohong Liu, Yang Bai, Wencong Xiao, Jianxi Ye, Minlan Yu, Hong Xu"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764848"
tags: ["query:dcn-comm"]
score: 8.0
evidence: 追踪LLM训练中集合通信状态与依赖关系
tldr: 集合通信库如同黑盒，隐藏了根因分析所需的关键信息，使LLM训练中的可靠性问题难以解决并浪费资源。Mycroft对集合通信状态进行轻量分布式追踪，并利用内部的控制与数据依赖关系定位故障根因。系统已在字节跳动部署超过六个月，能够揭示此前隐藏的可靠性问题。该工作提升了集合通信的可观测性，有助于保障大规模LLM训练的可靠性。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 现有集合通信库为黑盒，隐藏根因分析信息，导致LLM训练中的可靠性问题难以定位。
method: Mycroft对集合通信状态进行轻量分布式追踪，并利用内部控制与数据依赖关系诊断故障。
result: 系统在字节跳动部署逾六个月，可定位此前隐藏的集合通信可靠性问题。
conclusion: 该工作提升了集合通信可观测性，有助于保障大规模LLM训练的可靠性。
---

## Abstract
Reliability is essential for ensuring efficiency in LLM training. However, many real-world reliability issues remain difficult to resolve, resulting in wasted resources and degraded model performance. Unfortunately, today's collective communication libraries operate as black boxes, hiding critical information needed for effective root cause analysis. We propose Mycroft, a lightweight distributed tracing and root cause analysis system designed to address previously hidden reliability issues in collective communication. Mycroft's key idea is to trace collective communication states and leverage internal control and data dependencies to resolve reliability problems in LLM training. Mycroft has been deployed at ByteDance for over six months to debug collective communication-related issues at runtime. It detected anomalies within 15 seconds in 90% of cases and identified the root cause within 20 seconds in 60% of cases. We also conducted extensive fault injection experiments to demonstrate Mycroft's capability and efficiency.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
追踪LLM训练中集合通信状态与依赖关系。

### 2. 核心内容
集合通信库如同黑盒，隐藏了根因分析所需的关键信息，使LLM训练中的可靠性问题难以解决并浪费资源。Mycroft对集合通信状态进行轻量分布式追踪，并利用内部的控制与数据依赖关系定位故障根因。系统已在字节跳动部署超过六个月，能够揭示此前隐藏的可靠性问题。该工作提升了集合通信的可观测性，有助于保障大规模LLM训练的可靠性。

### 3. 对应检索需求
collective communication protocols and primitives for distributed systems。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764848](https://dl.acm.org/doi/10.1145/3731569.3764848)
