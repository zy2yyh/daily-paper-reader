---
title: "Mycroft: Tracing Dependencies in Collective Communication Towards Reliable LLM Training"
title_zh: Mycroft：面向可靠LLM训练的集合通信依赖追踪
authors: "Yangtao Deng, Lei Zhang, Xiaoyun Zhi, Lei Wang, Gaohong Liu, Yang Bai, Wencong Xiao, Jianxi Ye, Minlan Yu, Hong Xu"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3731569.3764848"
tags: ["query:dcn-comm"]
score: 8.0
evidence: 对集合通信状态与依赖进行分布式追踪以保障LLM训练
tldr: 集合通信库目前如同黑盒，隐藏了根因分析所需的关键信息，导致LLM训练中许多可靠性问题难以解决。Mycroft对集合通信状态进行追踪，利用内部控制与数据依赖关系定位可靠性问题，实现轻量级分布式追踪与根因分析。系统已在字节跳动部署超过六个月并验证有效。其贡献在于为集合通信的可观测性与可靠性保障提供了系统化工具。
source: SOSP-2025-ACM
selection_source: conference_retrieval
motivation: 集合通信库作为黑盒隐藏关键信息，使LLM训练中的可靠性问题难以定位。
method: 构建轻量级分布式追踪与根因分析系统，追踪集合通信状态并利用控制与数据依赖。
result: 系统在字节跳动部署超过六个月，有效解决此前难以发现的集合通信可靠性问题。
conclusion: 该工作提升了集合通信的可观测性，为可靠的大规模LLM训练提供支撑。
---

## Abstract
Reliability is essential for ensuring efficiency in LLM training. However, many real-world reliability issues remain difficult to resolve, resulting in wasted resources and degraded model performance. Unfortunately, today's collective communication libraries operate as black boxes, hiding critical information needed for effective root cause analysis. We propose Mycroft, a lightweight distributed tracing and root cause analysis system designed to address previously hidden reliability issues in collective communication. Mycroft's key idea is to trace collective communication states and leverage internal control and data dependencies to resolve reliability problems in LLM training. Mycroft has been deployed at ByteDance for over six months to debug collective communication-related issues at runtime. It detected anomalies within 15 seconds in 90% of cases and identified the root cause within 20 seconds in 60% of cases. We also conducted extensive fault injection experiments to demonstrate Mycroft's capability and efficiency.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
对集合通信状态与依赖进行分布式追踪以保障LLM训练。

### 2. 核心内容
集合通信库目前如同黑盒，隐藏了根因分析所需的关键信息，导致LLM训练中许多可靠性问题难以解决。Mycroft对集合通信状态进行追踪，利用内部控制与数据依赖关系定位可靠性问题，实现轻量级分布式追踪与根因分析。系统已在字节跳动部署超过六个月并验证有效。其贡献在于为集合通信的可观测性与可靠性保障提供了系统化工具。

### 3. 对应检索需求
collective communication protocols and primitives for distributed systems。

### 4. 来源与原文
- Source：SOSP-2025-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3731569.3764848](https://dl.acm.org/doi/10.1145/3731569.3764848)
