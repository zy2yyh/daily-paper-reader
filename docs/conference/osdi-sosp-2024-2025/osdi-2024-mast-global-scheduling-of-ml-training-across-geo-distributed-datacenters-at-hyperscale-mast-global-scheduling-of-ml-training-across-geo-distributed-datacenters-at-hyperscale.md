---
title: "{MAST}: Global Scheduling of {ML} Training across {Geo-Distributed} Datacenters at Hyperscale"
title_zh: MAST：超大规模地理分布式数据中心间机器学习训练的全局调度
authors: "Arnab Choudhury, Yang Wang, Tuomas Pelkonen, Kutta Srinivasan, Abha Jain, Shenghao Lin, Delia David, Siavash Soleimanifard, Michael Chen, Abhishek Yadav, Ritesh Tijoriwala, Denis Samoylov, Chunqiang Tang"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi24-choudhury.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 跨地理分布数据中心的机器学习训练全局调度
tldr: 该论文针对公有云中用户需手动选择区域放置训练数据与负载、导致跨区域供需错配的问题，在Meta超大规模私有云中提出全局调度抽象MAST。用户只需提交训练作业，由MAST智能地将数据与计算放置到不同区域。论文阐述三条设计原则并展示其在提升硬件利用率与盈利能力方面的效果，为超大规模数据中心资源调度提供了新范式。
source: OSDI-2024-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-mast-global-scheduling-of-ml-training-across-geo-distributed-datacenters-at-hyperscale/fig-001.webp\", \"caption\": \"\", \"page\": 1, \"index\": 1, \"width\": 875, \"height\": 356}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-mast-global-scheduling-of-ml-training-across-geo-distributed-datacenters-at-hyperscale/fig-002.webp\", \"caption\": \"\", \"page\": 1, \"index\": 2, \"width\": 875, \"height\": 360}]"
motivation: 公有云中用户各自选择区域易造成跨区域算力供需错配，降低云厂商硬件利用率与盈利。
method: 提出面向所有ML训练作业的全局调度抽象MAST，自动将数据与训练负载放置到不同区域。
result: 论文给出三条设计原则，并在Meta超大规模私有云中验证了全局调度对利用率与盈利的改善。
conclusion: 全局调度可缓解跨区域供需错配，为超大规模云中训练负载放置提供了可行方案。
---

## Abstract
In public clouds, users must manually select a datacenter region to upload their ML training data and launch ML training workloads in the same region to ensure data and computation colocation. Unfortunately, isolated decisions by individual users can lead to a mismatch between workload demand and hardware supply across regions, hurting the cloud provider's hardware utilization and profitability. To address this problem in Meta's hyperscale private cloud, we provide a global-scheduling abstraction to all ML training workloads. Users simply submit their training workloads to MAST, our global scheduler, and rely on it to intelligently place both data and training workloads to different regions. We describe three design principles that enable MAST to schedule complex ML training workloads at a global scale: temporal decoupling, scope decoupling, and exhaustive search. MAST successfully balances the load across global regions. Before MAST, the most overloaded region had a GPU demand-to-supply ratio of 2.63 for high-priority workloads. With MAST, this ratio has been reduced to 0.98, effectively eliminating the overload.

---

## 论文详细总结（自动生成）

# MAST 论文中文总结

## 1. 核心问题与整体含义

- **研究背景**：ML 训练已成为数据中心增长最快的负载之一。公有云和 Meta 私有云通常拥有多个地理分布的数据中心区域（region），用户需要手动选择区域上传训练数据，并在同一区域启动训练作业，以保证数据与计算同置。
- **核心问题**：用户各自独立选择区域，容易造成跨区域的工作负载需求与硬件供给错配：某些区域 GPU 已耗尽并堆积长队列，另一些区域却有闲置容量。Meta 私有云中，高优先级负载在过载最严重区域的 GPU 需求/供给比曾高达 **2.63**。
- **整体含义**：论文提出 **全局调度抽象（global-scheduling abstraction）**，用户只需把 ML 训练负载提交给全局调度器 **MAST**，由 MAST 联合决定训练数据和计算负载在不同区域间的放置，从而提升硬件利用率与云厂商盈利能力。
- **核心贡献**：提出全局调度抽象；总结三条设计原则——**时间解耦、范围解耦、穷举搜索**；并在 Meta 超大规模私有云中部署验证。

## 2. 方法论

### 2.1 核心思想：三条设计原则

- **时间解耦（temporal decoupling）**：将调度分为慢路径和快路径。慢路径在后台持续优化数据放置和机器分配；快路径负责实时作业调度。慢路径通过跨区域复制训练数据，使快路径更容易实现数据与 GPU 同置。
- **范围解耦（scope decoupling）**：把传统上集中在集群范围的三个职责拆分到不同范围：
  - **GMS（Global ML Scheduler）**：全局作业队列管理；
  - **RMS（Regional ML Scheduler）**：区域级资源分配；
  - **CM（Cluster Manager）**：集群级容器编排。
- **穷举搜索（exhaustive search）**：ML 训练作业运行时间长、GPU 昂贵，不应像短时分析作业那样用简单启发式只搜索一个集群；MAST 在多个有数据且有硬件的区域/动态集群中并行计算放置方案，再通过拍卖选择最优方案。

### 2.2 慢路径：Tetris 数据放置

- **问题建模**：Tetris 将表到区域的放置建模为混合整数规划（MIP）。决策变量主要是每张表的 home region；作业所在区域由表放置推导。
- **优化目标**：
  - 最小化训练时跨区域读取的数据量；
  - 软平衡各区域 GPU 需求与供给；
  - 软平衡各区域存储需求与供给。
- **硬约束**：
  - 每区域存储容量约束；
  - 应用 CPU 配额约束；
  - 高优先级作业满足“在 home region 训练”的比例不低于阈值。
- **GPU 特殊处理**：由于 GPU 稀缺，若像 CPU/存储那样强加“每区域需求不超过供给”的硬约束，MIP 常无解。MAST 允许 GPU 超卖，并按需抢占低优先级作业；目标函数中对 GPU 过载使用平方偏差加 sigmoid 惩罚，且过载惩罚重于欠载。
- **求解算法**：使用爬山式局部搜索近似求解；每日运行一次，约 5 小时；通过移动队列按收益排序迁移表，受每日跨区域迁移配额限制。
- **额外副本**：为灾备、热表、以及无法在单一区域同时满足数据和 GPU 类型的作业创建额外副本；存储开销增加约 **75%–125%**。

### 2.3 快路径：GMS → RMS → CM

- **GMS**：维护全局待调度队列，为每个负载计算 `<priority, credit>` 元组。优先级受租户配额影响；超出配额的负载降为最低优先级、可被抢占。credit 综合等待时间和租户公平份额。GMS 周期性执行 GMS-scan 更新所有待调度负载的元组。
- **RMS**：各区域 RMS 以分布式拍卖方式调度。RMS 检查本区域是否有必需数据和硬件类型；若有，则为负载中每个作业在每个 ML 动态集群中计算放置方案及质量分 **Pscore**，选择最高分方案。区域间通过拍卖选出最高 Pscore 的 RMS 执行。
- **Pscore 与抢占**：不抢占即可运行的方案得分更高；必须抢占时，优先抢占低优先级、数量少、运行时间长的作业，避免抢占新启动作业。RMS 还使用**负缓存**过滤大量注定失败的分配尝试，生产命中率约 **80%**。
- **CM**：Twine 集群管理器负责容器编排；RAS 慢路径预先将机器分配给 ML/非 ML 动态集群，RMS 只需搜索 ML 动态集群，从而提升可扩展性。
- **绑定策略**：MAST 采用 **late binding**，只有当确认某集群可立即运行作业时才派发，而非传统 federation 的 early binding。

## 3. 实验设计

- **实验类型**：论文不是传统意义上的数据集/模型精度实验，而是基于 Meta 生产部署、生产 trace 和模拟器的系统评估。
- **场景与规模**：
  - 数十个区域、数百万台机器、约 **O(100,000) GPU** 和 **O(100,000) CPU 机器**；
  - 每天调度数万个 ML 训练负载；
  - 约 70% 负载使用 GPU，约 30% 为 recurring 负载，约 70% 为首次负载；
  - 平均每个负载被抢占约 1 次，每天约 10 万次调度尝试。
- **主要评估指标**：
  - 两周内的 GPU 分配率；
  - 因数据-GPU 不同置而等待数据的负载比例；
  - 30 天内每个负载的候选区域数与实际执行区域数；
  - Tetris 每日跨区域数据移动量；
  - GMS-scan 延迟、最慢 RMS 的调度延迟；
  - 负载从 pending 到 running 的平均/P95 延迟；
  - 模拟器中 SLO 违反率。
- **对比方法**：
  - **Tetris V0/V1/V2**：V0 为人工选区域且表不可迁移；V1 自动化数据放置；V2 改进 GPU 过载惩罚、存储软平衡和同置率定义。
  - **Fast-path 模拟器对比 federated 方法**：federated 将负载派发到所需 GPU 类型 demand/supply 最低的区域；使用 8 小时生产 trace。
  - **Yugong**：因强制硬配额且不考虑 GPU，在 GPU 持续供不应求的环境中不可比；**Singularity** 因调度细节未披露无法具体比较。

## 4. 资源与算力

- 论文披露的是**生产调度规模**，不是模型训练实验的算力配置：
  - 数十个区域、数百万台机器；
  - 约 **10 万级 GPU** 和 **10 万级 CPU 机器**；
  - 最大 RMS 管理约 **20 个动态集群**，合计 **64,000 台 CPU 机器**和 **20,000 块 GPU**；
  - 最大单负载为 LLM 预训练，使用**数万块 GPU**。
- **训练时长分位数**：GPU 负载运行时长 P50/P90/P99 约为 **20 分钟 / 6.7 小时 / 66 小时**；CPU 负载约为 **38 分钟 / 7.9 小时 / 38 小时**。
- **Tetris 开销**：每日在单机上运行约 **5 小时**，每天跨区域移动**数百 PB** 数据。
- **未明确说明**：GPU 具体型号、总训练时长、能耗、成本、集群网络拓扑细节等未披露；论文也没有进行模型训练精度或收敛性实验。

## 5. 实验数量与充分性

- **实验组数**：大致包括 6 类生产测量（GPU 分配率、数据不同置率、候选/执行区域分布、每日数据移动、GMS 延迟、RMS 延迟与调度延迟），加上 Tetris V0/V1/V2 版本对比和 fast-path 模拟器对比。
-
