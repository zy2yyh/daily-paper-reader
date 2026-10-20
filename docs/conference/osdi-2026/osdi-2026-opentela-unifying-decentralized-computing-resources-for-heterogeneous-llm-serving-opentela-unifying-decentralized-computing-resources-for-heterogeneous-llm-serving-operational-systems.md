---
title: "{OpenTela}: Unifying Decentralized Computing Resources for Heterogeneous {LLM} Serving (Operational Systems)"
title_zh: OpenTela：统一去中心化计算资源以支持异构LLM服务
authors: "Xiaozhe Yao, Youhe Jiang, Ilia Badanin, Qinghao Hu, Robert Matthew Smith, Binhang Yuan, Imanol Schlag, Eiko Yoneki, Ana Klimovic"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-yao.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 在大规模HPC集群上提供路由与负载均衡的去中心化控制面
tldr: 主权AI希望在自控的HPC集群上服务大模型，但这类集群按批处理作业优化，缺少服务发现、路由、健康监测与负载均衡等控制面能力，资源分配还是瞬态的。OpenTela构建统一去中心化资源层，在Slurm等调度环境中提供跨集群的服务发现、路由与负载均衡。评测显示其能在异构且资源瞬态可用的集群上完成LLM服务的请求调度与路由。该工作说明去中心化控制面可让HPC集群承担常驻推理服务。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-opentela-unifying-decentralized-computing-resources-for-heterogeneous-llm-serving/fig-001.webp\", \"caption\": \"\", \"page\": 10, \"index\": 1, \"width\": 867, \"height\": 867}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-opentela-unifying-decentralized-computing-resources-for-heterogeneous-llm-serving/fig-002.webp\", \"caption\": \"\", \"page\": 10, \"index\": 2, \"width\": 867, \"height\": 867}]"
motivation: 主权AI希望在自有HPC集群上服务大模型，但这类集群缺少服务发现、路由、健康监测与负载均衡等控制面能力。
method: 构建统一去中心化资源层，在Slurm管理的瞬态分配环境中提供跨集群的服务发现、路由与负载均衡。
result: 在异构且资源瞬态可用的HPC环境中实现可用的LLM服务调度与请求路由。
conclusion: 说明去中心化控制面可让HPC集群承担常驻的异构LLM推理服务。
---

## Abstract
Large language models (LLMs) are becoming critical to a variety of public services, motivating sovereign AI initiatives that seek to serve models on infrastructure they control. Yet much of this infrastructure is built as HPC clusters optimized for batch jobs rather than interactive, always-on inference services. Existing LLM serving engines efficiently execute requests on GPUs, but rely on an external control plane (e.g., Kubernetes in cloud environments) for service discovery, routing, health monitoring, and load balancing. In HPC environments, these primitives are often unavailable: resources are managed by schedulers such as Slurm, allocations are transient, compute nodes may be unreachable from outside the cluster, and GPU capacity is fragmented across heterogeneous clusters and administrative domains. We present OpenTela, a user-space orchestration overlay that turns existing fragmented HPC clusters into a unified, cross-institutional serving platform. OpenTela provides fault-tolerant service discovery via a CRDT-based gossip network, a unified API over heterogeneous cluster managers and serving engines, and a heterogeneity-aware scheduler, all in user-space without root privileges or cluster reconfiguration. OpenTela has been deployed for over 22 months, serving 13 million requests and 15 billion tokens across 142 models to over 1000 researchers across multiple institutions. We open-source the system and release an anonymized production trace to facilitate further research into real-world LLM serving workloads, and provide a replicable blueprint for other sovereign AI initiatives to harness their own federated GPU infrastructure.

---

## 论文详细总结（自动生成）

# OpenTela 论文结构化总结

## 1. 核心问题与整体含义

### 研究动机
- **主权 AI（Sovereign AI）浪潮**：LLM 已成为行政、医疗、教育等公共服务的关键基础设施，政府与机构希望模型运行在自己可控、可审计、可治理的基础设施上，而非依赖商业 API（无法知晓训练偏差、无法控制模型可用性与版本演进）。
- **大规模 HPC 投资**：瑞士 Alps（10,000+ GH200）、日本 ABCI 3.0（6,128 H200）、欧盟 19 个 AI Factories（200 亿欧元）、英国 20 亿英镑等，算力以 HPC 集群形式落地。
- **核心矛盾**：HPC 集群为**吞吐导向的批处理作业**设计（Slurm 管理、固定 wall-clock 分配、作业即瞬态进程），而 LLM 推理是**常驻、交互式服务**，需要持久端点、服务发现、请求路由、负载均衡、健康监测与弹性伸缩。
- **执行层 ≠ 服务平台**：vLLM / SGLang 只负责模型加载、KV cache、连续批处理与调度，默认依赖外部控制面（云上 Kubernetes）提供上述能力；HPC 环境中这些原语缺失。

### 现实约束
- 用 Kubernetes 替换 Slurm 不现实：HPC 负载依赖拓扑感知调度、原生 MPI 启动与既有运维策略。
- 划分独立 K8s 分区会刚性割裂稀缺 GPU，导致一侧闲置、另一侧排队。
- GPU 可用性高度动态（图 1 显示 ETH 集群空闲 GPU 比例随时间和型号剧烈波动），推理负载突发性强，因此**跨集群池化**有明确收益。
- 但跨集群联邦面临：不同管理机构、不同调度器、不同网络策略、异构 GPU、不同数据主权要求；计算节点位于 NAT 之后、仅有出站连通性；**不存在所有站点都愿意/能够运营的中央特权协调者**。

### 整体含义
- OpenTela 是一个**纯用户态编排覆盖层（overlay）**，在既有 Slurm / Kubernetes 之上把碎片化、跨机构的 HPC 集群统一为单一 LLM 服务平台，无需 root 权限、无需集群重配置、不改变用户工作流。
- 已在瑞士 AI 计划生产运行 22 个月以上，服务 13M 请求、15B token、142 个模型、1000+ 研究者，并提供可复制的"主权 AI 利用自有联邦 GPU 基础设施"蓝图。

---

## 2. 方法论

### 2.1 核心思想
- 不替换底层集群管理器，而是**旁路叠加**：provider 在 Slurm allocation 或 K8s deployment 上启动 OpenTela 进程，包裹现有推理引擎命令（如 `otela start --process vllm serve <model>`），节点加入 P2P 网络并广播硬件与模型元数据；consumer 通过统一 API 提交请求。

### 2.2 四条设计需求
- **R1 非侵入部署**：普通用户权限，不改调度器、不改集群配置。
- **R2 瞬态资源上的服务抽象**：把短时分配与独立启动的进程变成持久服务（发现、注册、健康、负载均衡、路由），容忍节点加入/离开/失败/被抢占。
- **R3 无中心协调者的联邦**：跨独立管理域，最终一致视图，容忍网络分区与节点抖动，无单点故障。
- **R4 异构与信任感知路由**：支持多种 GPU、引擎、模型变体；请求只路由到用户/机构授权的 provider。

### 2.3 系统组件
- **统一服务协议**：以 OpenAI 兼容 API 为基础（引擎无关），扩展只读的基础设施端点（实时节点状态、模型注册表、provider 元数据），严格只读以保护分布式状态完整性。
- **节点网络（libp2p）**：提供对等寻址与发现、加密通道、NAT 穿透；通过 rendezvous 打洞或公网 peer 的 circuit relay 建立连接，仅使用节点已获准的出站连接；bootstrap peer + 指数退避重试；Kademlia DHT 发现 peer；在其上叠加**随机化 gossip** 传播状态。
- **注册表（CRDT）**：采用 **G-Map（Growth-only Map）CRDT**，merge 语义满足可交换、可结合、幂等，无需协调即可收敛；key 为节点唯一 ID，value 为生命周期状态 + 元数据；每个节点持有全量本地副本、本地读、无网络往返。
  - 两个视图：**Node Registry**（硬件与生命周期）、**Model Registry**（模型 → 服务节点映射）。
  - **两级垃圾回收**：先在保留窗口（默认 24h）后从应用视图移除，再周期性压缩 CRDT tombstone，防止无界增长。
- **API frontend / ingress node**：以公网可达的普通 OpenTela 节点身份运行，持有完整 CRDT 副本，本地完成候选选择与路由；负责认证、用量统计、请求-响应生命周期；用户面服务与 ingress 可分离部署。
- **服务调度器**：集中式组件，做模型放置优化，策略可插拔（可由运维替换）。
- **Web 仪表盘**：浏览实时模型目录与承载硬件、管理 API key、交互式对话验证。

### 2.4 信任模型
- **准入控制**：只有维护者签名的 OpenTela 二进制可通过 join 握手，未签名/被篡改者不进入 CRDT 注册表。
- **客户端过滤**：请求元数据可携带可信 provider ID 白名单，ingress 在负载均衡前裁剪候选集，**绝不向未显式授权的 provider 派发请求**。
- **明确承认的缺口**：软件级签名无法阻止 provider（节点 root）替换推理引擎或在 loopback 抓取 prompt；提出以机密计算 + 远程证明作为未来方向。

### 2.5 生命周期与请求流程
- **节点状态严格有序**：`JOIN ≺ SERVING ≺ DOWN ≺ LEFT`；merge 函数取最高生命周期状态，网络自动收敛到最新完整配置。
  - `DOWN` 由本地健康检查发现推理子进程终止触发；
  - `LEFT` 为永久驱逐，需超过心跳宽限期；采用**三路独立探测全部失败**才判定"可疑"，可疑节点可在超时前广播存活更新自证；被驱逐节点以新 session ID 重新 JOIN。
- **请求路径**：ingress 查本地副本 → 按模型筛候选 → 按 provider 白名单过滤 → 选择 session ID（默认均匀随机，可选 round robin、按 GPU 等级加权随机）→ 派发到该节点 API 端点。
- **容错**：非流式请求对失败做透明重路由重试（有上限）；**流式请求**因推理非确定性、无法从任意续点重放，需客户端自行重试。
- **策略可插拔接口**：`Pick` / `OnRequestStart` / `OnRequestEnd` 三方法；fleet 级控制循环可轮询任一节点获取全局空闲容量并决定部署（论文示例：将 Helix 的 max-flow 选择映射到 `Pick`，放置决策映射到 fleet 控制器）。

### 2.6 放置优化（默认策略）
- **目标**：在异构资源池上最小化平均端到端时延
  `min_{A,P} L(W, A, P)`
  -
