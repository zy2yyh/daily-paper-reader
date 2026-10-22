---
title: DNS Congestion Control in Adversarial Settings
title_zh: 对抗环境下的DNS拥塞控制
authors: "Huayi Duan, Jihye Kim, Marc Wyss, Adrian Perrig"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://dl.acm.org/doi/pdf/10.1145/3694715.3695982"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向服务器间通道的公平排队调度与拥塞控制框架
tldr: 论文研究DNS场景中的对抗性拥塞攻击：攻击者只需低成本地阻塞服务器间通道，即可让大量用户无法访问目标DNS服务。作者借鉴经典网络拥塞控制，提出DNS拥塞控制框架DCC，通过优化的公平排队消息调度保证正常客户端公平访问通道，并结合可扩展的异常检测与信令机制减少对无害客户端的连带影响。该框架以非侵入方式设计，可增强现有DNS服务。其调度与拥塞控制思路对集合通信性能优化有方法层面的借鉴意义。
source: SOSP-2024-ACM
selection_source: conference_retrieval
motivation: 攻击者可低成本阻塞服务器间通道，造成大规模用户无法访问目标DNS服务。
method: 提出DCC框架，结合公平排队消息调度与异常检测信令机制进行非侵入式缓解。
result: 在攻击存在时仍能保证正常客户端公平访问服务器间通道。
conclusion: 为对抗性拥塞提供了通用调度思路，可迁移到其他网络通信场景。
---

## Abstract
We instigate the study of adversarial congestion in the context of the Domain Name System (DNS). By strategically choking inter-server channels, this new type of DoS attack can disrupt a large user group's access to target DNS servers at a low cost. In reminiscence of classic network congestion control, we propose a DNS congestion control (DCC) framework as a fundamental yet practical mitigation measure for such attacks. With an optimized fair-queuing message scheduler, DCC ensures benign clients fair access to inter-server channels regardless of an attacker's behavior; with a set of extensible anomaly detection and signaling mechanisms, it minimizes collateral damage to innocuous clients. We architect DCC in a non-invasive style so that it can readily augment existing DNS servers. Our prototype evaluation demonstrates that DCC effectively mitigates adversarial congestion while incurring minor performance overheads.

---

## 论文详细总结（自动生成）

### 1. 检索相关性
面向服务器间通道的公平排队调度与拥塞控制框架。

### 2. 核心内容
论文研究DNS场景中的对抗性拥塞攻击：攻击者只需低成本地阻塞服务器间通道，即可让大量用户无法访问目标DNS服务。作者借鉴经典网络拥塞控制，提出DNS拥塞控制框架DCC，通过优化的公平排队消息调度保证正常客户端公平访问通道，并结合可扩展的异常检测与信令机制减少对无害客户端的连带影响。该框架以非侵入方式设计，可增强现有DNS服务。其调度与拥塞控制思路对集合通信性能优化有方法层面的借鉴意义。

### 3. 对应检索需求
What network scheduling and congestion control techniques improve collective communication performance。

### 4. 来源与原文
- Source：SOSP-2024-ACM
- OpenReview：[https://dl.acm.org/doi/10.1145/3694715.3695982](https://dl.acm.org/doi/10.1145/3694715.3695982)
