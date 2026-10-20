---
title: "Efficient {LLM} Serving on Commodity {GPU} Clusters with {Data-Reduced} {Cross-Instance} Orchestration"
title_zh: 在通用GPU集群上通过数据缩减的跨实例编排实现高效LLM服务
authors: "Jiangsu Du, Hongbin Zhang, Taosheng Wei, Zhenyi Zheng, Jiazhi Jiang, Kaiyi Wu, Zhiguang Chen, Yutong Lu"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-du.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向通用GPU集群互联的跨实例编排
tldr: 现有LLM服务策略要么存在预填充与解码干扰，要么依赖通用GPU集群所缺乏的高性能互联。EcoServe面向通用GPU集群提出部分解耦策略，通过减少数据传输的跨实例协作缓解阶段干扰。系统在主流AI基础设施上实现了更优的服务效率。其贡献在于为缺乏高速互联的集群提供了适配的推理通信编排方法，但与集合通信算法关系有限。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 通用GPU集群缺乏高性能互联，现有LLM服务策略难以兼顾阶段干扰与互联约束。
method: 提出部分解耦策略，通过减少数据传输的跨实例协作来缓解预填充与解码干扰。
result: 在主流通用GPU集群上实现更优的推理服务效率与资源利用。
conclusion: 该工作为受限互联集群中的LLM服务提供了适配的跨实例编排方案。
---

## Abstract
Existing LLM serving strategies can be categorized by whether prefill and decode phases are disaggregated: non-disaggregated (NoDG) or fully disaggregated (FuDG). However, they neither fit commodity GPU clusters, which remain widely deployed as mainstream AI infrastructure. NoDG suffers from severe prefill–decode interference, while FuDG depends heavily on high-performance interconnects that such clusters lack. We present EcoServe, an LLM serving system tailored to commodity GPU clusters. It enables a data-reduced collaboration among inference instances to mitigate prefill-decode interference, termed the partially disaggregated (PaDG) strategy. Particularly, within a single instance, PaDG disaggregates the prefill and decode phases along the time dimension to mitigate interference and enhance throughput. Next, it coordinates multiple instances and cyclically activates them to ensure the continuous availability of prefill processing, thereby rescuing latency. Thus, EcoServe’s basic serving unit is the macro instance, within which multiple instances collaborate. It further incorporates an adaptive scheduling algorithm to route requests in a macro instance and a mitosis scaling approach for fine-grained capacity adjustments in online scenario. On a 32-GPU NVIDIA L20 cluster over Ethernet, EcoServe improves goodput by 1.96×, 1.99×, 2.51×, and 2.40× when serving 30B- and 70B-scale LLMs, compared to four representative NoDG and FuDG systems, vLLM, Sarathi, DistServe, and MoonCake. EcoServe remains competitive even on an NVIDIA H100 cluster with NVLink and InfiniBand. Our code is released at https://github.com/MLSysU/EcoServe .

---

## 论文详细总结（自动生成）

# EcoServe 论文结构化总结

## 1. 核心问题与研究背景

- **现实约束**：尽管 H100/GB200 等高端 GPU 被广泛宣传，但受成本与生产环境更新周期限制，**缺乏高性能互联（NVLink/InfiniBand）的通用 GPU 集群仍是主流 AI 基础设施**。论文作者在腾讯某超级 App 的 L20 集群上开展研究，具有真实生产背景。
- **LLM 推理的双阶段特性**：prefill 计算密集（compute-bound），decode 访存密集（memory-bound），分别对应 TTFT 与 TPOT 两个时延 SLO，加上吞吐构成“性能权衡三角”。
- **现有两类策略均不适配通用集群**：
  - **NoDG（非解耦）**：prefill 与 decode 同实例共置，产生严重阶段干扰；decode 难以积累足够大的 batch 以打满 GPU；与流水线并行兼容性差（负载不均、微批依赖导致流水气泡）。
  - **FuDG（完全解耦）**：将两阶段分置不同实例，需在实例间传输海量 KV cache。文中实测 KV cache 生成速率：Llama-30B 在 L20 上 9.796 GB/s、A800 上 38.96 GB/s、H100 上 95.54 GB/s，A800 场景至少需 400 Gbps 网络；此外还存在计算/内存双重负载不均、并行兼容性受限、工程复杂度高三重问题。
- **核心命题**：需要一种**不依赖高速互联、以数据缩减方式缓解 prefill–decode 干扰**的集群级服务策略。

## 2. 方法论：PaDG（部分解耦）策略

### 2.1 核心思想

- 关键洞察：**实例内调度（何时执行 prefill/decode）应与实例间调度（何时、向何处路由请求）协同**，才能抬高权衡三角的上界。
- PaDG 通过“时间维度解耦 + 滚动激活”在两个层级上主动调度，实现**数据缩减（data-reduced）的跨实例协作**：既保留解耦带来的干扰抑制，又避免跨实例 KV cache 传输。

### 2.2 时间维度解耦（Temporal Disaggregation）

- 不将 prefill/decode 分配给不同实例，而是**映射到同一实例的不同时间片**，且每个阶段持续时间更长。
- 效果：降低阶段切换频率 → 缓解干扰、提升吞吐；KV cache 始终本地保留 → 无跨实例数据搬运，提升片上资源利用率。
- **利用“节省的 TPOT”（saved TPOT）**：现代服务按“打字机”方式输出，只要时间窗内产出足够 token 即满足 TPOT；若 decode 快于 TPOT 约束，便积累松弛时间，可用于插入 prefill 而不违反 TPOT。
- 代价：单实例在 decode 期间到达的新请求需等待下一个 prefill 窗口，可能违反 TTFT——由滚动激活解决。

### 2.3 滚动激活（Rolling Activation）与宏实例

- 多个实例**以循环方式错峰进入 prefill 阶段**，保证任意时刻至少有一个实例在处理 prefill，使新请求总能被立即接纳，从而保住 TTFT。
- 协作实例组称为**宏实例（macro instance）**，是 EcoServe 的基本服务单元。
- 由于输出长度未知，实例需持续向宏实例调度器上报 decode 进度、内存占用等状态。

### 2.4 分层架构与自适应调度算法

- **三级调度**：
  - *实例调度器*：管理单实例内 prefill–decode 协调、设备编排，执行上层指令；策略上**优先 prefill**。
  - *宏实例调度器*：聚合各实例状态，依据 profiling 结果与 SLO 派发请求；采用**主–从（master–worker）**模式。
  - *总体调度器*：按能力将请求分配给各宏实例。
- **实现支撑**：同节点上构建公共消息队列供实例周期上传状态，ZeroMQ 同步状态，Ray 控制多设备，vLLM 0.7.3 作为单设备运行时。
- **算法 1（约束检查）流程**（文字化）：
  1. **TTFT 约束**：计算本轮待处理 prefill 的总时长 `t_total`（由 profiling 预测量）；若 `t_total > S_TTFT` → 不满足。
  2. **TPOT 约束**：对每个已存在的 decode 请求，`saved_tpot = L × S_TPOT − (当前时刻 − 首 token 时刻)`；取均值，若 `mean_saved_tpot < t_total` → 不满足。
  3. **内存约束**：若请求 KV cache 大小 > 剩余显存 → 不满足。
  4. 全部通过则接纳。
- **调度遍历方式**：循环遍历实例，优先尝试路由到处理上一个请求的同一实例，不满足则检查下一个。
- **鲁棒性设计**：松弛量在**短解码区间**内估计、本地累积、由新到 prefill 消耗，不依赖长期预测。

### 2.5 有丝分裂扩缩容（Mitosis Scaling）

- 动机：以宏实例为粒度扩缩容过于粗糙；单一宏实例管理全部实例会形成调度瓶颈。
- **两个超参**：`Nl`（宏实例内实例数下界，过小导致频繁阶段切换）、`Nu`（上界，过大导致管理开销与调度瓶颈）。
- **扩容流程**：在宏实例内增量加实例（步 1）→ 超过 `Nu` 时分裂出一个含 `Nl` 个实例的新宏实例（步 2）→ 继续加至 `Nu`（步 3）→ 后续实例加入新宏实例（步 4）。
- **缩容流程**：从最小宏实例移除至 `Nl`（步 5）→ 从满宏实例移除（步 6）→ 两者总数达 `Nu` 时合并（步 7–8）；被移除实例先完成已分配请求再释放。
- **扩缩容触发阈值**：
  - 扩容信号 = **TTFT SLO 达成率**（过载时最先、最敏感地下降，因无实例满足约束时请求进入 pending），判据为平均 TTFT 超过 `S_TTFT`（式 1）。
  - 缩容信号 = **saved TPOT**；合理区间为 `[T_TTFT, T_TTFT × (n+1)/n]`，超过上界即判定过配（式 2）。
- **灵活实例迁移**：设计**可序列化代理对象** `InstanceHandler`（封装实例入口点与公共消息队列），用 pickle 序列化后由总体调度器协调转交给目标宏实例调度器，反序列化后接管控制权，实现**不中断执行的逻辑迁移**（对比：重新初始化 CodeLlama2-34B 从 L20 本地存储需约 3 分钟，远程更久）。
- **指标定义**：文中 TTFT 包含“阶段切换等待时间”，构成更严格的 SLO；TPOT 从阶段切换后才开始计时。

## 3. 实验设计

### 3.1 测试集群（3 个）

| 集群 | 节点/GPU | 互联 | 定位 |
|---|---|---|---|
| L20 | 8 节点 / 64× L20-48GB | PCIe + 10 Gbps 以太网 | 主战场（通用） |
| A800 | 2 节点 / 16× A800-80GB | PCIe + 25 Gbps RoCE | 通用（算力/带宽比更极端） |
| H100 | 2 节点 / 16× H100-80GB | NVLink + 8×400 Gbps IB | 高性能对照 |

### 3.2 模型、数据集与 SLO

- **模型**：Llama-30B（MHA）、CodeLlama2-34B（GQA）、Qwen2-72B（GQA），BF16。
- **数据集/SLO**：Alpaca-gpt4（短入长出，1s/100ms）、ShareGPT（均衡，5s/100ms）、LongBench（长入短出，15s/100ms）；SLO 仅由应用决定、与模型规模无关，多数比先前工作更严格。
- **负载**：固定速率 + Poisson 到达；突发场景用 Gamma 过程（CV=1.2/1.5/2.0）。作者扩展了 vLLM 请求发送程序以支持多线程高并发。

### 3.3 对比基线（均对齐 vLLM 0.7.3 以保证公平）

- **vLLM**：NoDG，分离批处理
