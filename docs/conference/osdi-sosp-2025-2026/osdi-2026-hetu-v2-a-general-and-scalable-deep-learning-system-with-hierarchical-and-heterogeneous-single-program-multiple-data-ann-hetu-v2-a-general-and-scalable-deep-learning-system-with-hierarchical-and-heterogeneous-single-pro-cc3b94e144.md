---
title: "Hetu v2: A General and Scalable Deep Learning System with Hierarchical and Heterogeneous Single Program Multiple Data Annotations"
title_zh: Hetu v2：具有分层异构SPMD标注的通用可扩展深度学习系统
authors: "Haoyang Li, Fangcheng Fu, Hao Ge, Sheng Lin, Xuanyu Wang, Jiawen Niu, Yuming Zhou, Xupeng Miao, Bin Cui"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-li-haoyang.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: SPMD抽象自动推导张量分片与通信模式以支持分布式训练
tldr: SPMD范式虽为分布式深度学习提供统一并行抽象，但大规模训练中设备差异与数据变化带来时空负载异构，违背其对称划分假设。Hetu v2提出分层且异构的SPMD标注，使系统能自动推导张量分片与通信模式，并适配不均衡负载。该系统提升了异构环境下的可扩展性与通用性。其贡献在于为分布式训练中的通信与并行规划提供了更灵活的系统抽象。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 大规模分布式训练存在设备与数据导致的时空负载异构，违背SPMD对称划分假设。
method: 提出分层且异构的SPMD标注，使系统自动推导张量分片与通信模式并适配不均衡负载。
result: 系统在异构环境下提升可扩展性与通用性，支持混合硬件与数据变化场景。
conclusion: 该工作为分布式深度学习的并行与通信规划提供了更灵活的系统抽象。
---

## Abstract
The Single-Program Multiple-Data (SPMD) paradigm provides a unified abstraction to annotate various parallel dimensions in distributed deep learning (DL) training. With SPMD, users can write training programs from the viewpoint of a single device, and the system will automatically deduce the tensor sharding and communication patterns. However, with the recent development in large-scale DL models, distributed training exhibits spatial and temporal workload heterogeneity, arising from both device disparities (e.g., mixed hardware, failures) and data variations (e.g., uneven sequence lengths). Such heterogeneity violates SPMD’s assumption of symmetric workload partitioning, which restricts its ability to express and optimize heterogeneous parallel strategies effectively. To address this, we propose HSPMD within the Hetu v2 system to achieve general and scalable DL training. HSPMD extends SPMD’s declarative annotations to support asymmetric sharding and composes standard communication primitives for hierarchical communication, all while retaining the simplicity of a single-device programming model. HSPMD handles spatial heterogeneity through progressive graph specialization, enabling device-specific execution logic, and addresses temporal heterogeneity via dynamic graph switching. Evaluations on (a) heterogeneous devices, (b) unstable devices, and (c) mixed-length data scenarios show that HSPMD matches or outperforms specialized systems, providing a flexible and efficient solution for modern distributed DL training. Code is available: https://github.com/PKU-DAIR/Hetu .

---

## 论文详细总结（自动生成）

# Hetu v2 / HSPMD 论文中文结构化总结

---

## 一、核心问题与整体含义（研究动机与背景）

- **SPMD 是分布式深度学习的主流抽象**：用户以"单设备视角"编写程序，通过声明式标注（如 Split/Duplicate、ColumnParallelLinear 等）让系统自动推导张量分片与通信模式，实现"并行策略与模型解耦"。GSPMD、Alpa、DTensor、DeepSpeed、Megatron 均属此列。
- **SPMD 的隐含前提是对称性**：(i) 硬件同构且稳定；(ii) 各数据带来的训练负担等价。其原语（all-reduce/all-gather/reduce-scatter、规则 device mesh 上的均匀切分）本质要求设备对称参与。
- **现实中的异构打破了该前提**，论文将其归纳为两个正交维度：
  - **空间异构（spatial）**：某一时刻的负载不均衡，需要"空间异构并行策略"。来源包括：GPU 短缺导致的混合代际集群（A100/H100 与低端卡混布，算力、显存、带宽不一）、设备/节点故障（Llama 3 训练 54 天发生 419 次中断，148 次源于 GPU 故障）、变长数据（文本长度、图像分辨率、音视频时长差异）。
  - **时间异构（temporal）**：负载随训练步演化，需要在不同并行策略间动态重构（如故障发生后重配 TP/PP 度；某一步骤出现超长序列需切换到长序列策略以避免 OOM）。
- **现有方案的局限**：
  - **MPMD**（Pathways、Ray）：为不同设备生成不同程序，单任务内部异构时需管理成千上万个程序，编译开销大、易用性差，可扩展性不足。
  - **调度器增强的 SPMD 变体**（HexiScale、Oobleck、HotSPa）：在 SPMD 之上外挂场景专用调度器，需绕过对称性约束、引入大量控制/分支逻辑；与特定工作负载紧耦合，**只能服务单一场景**，无法作为通用底座。
- **论文主张**：不应在调度器层做场景定制，而应把可扩展性下沉到**声明式标注原语层**，让 SPMD 自身原生支持非对称分片与通信，从而用统一抽象覆盖多类异构场景。这就是 Hetu v2 中的 **HSPMD（Hierarchical and Heterogeneous SPMD）**。

---

## 二、方法论

### 2.1 总体框架

- 输入：用户定义的**单一计算图**（defined graph，含 Leaf 算子、HSPMD 新增的 **Reshard** 抽象标记算子、普通计算算子）+ **annotation plan**（标注计划，用户或外部 planner 给出，仅标注 Leaf 与 Reshard 的输出）。
- 系统流程：标注推导 → 通信解析 → 为每设备生成**可执行图**（graph specialization，应对空间异构）→ 需要时在可执行图之间**在线切换**（graph switching，应对时间异构）。
- 两个抽象彼此正交、可组合，分别映射到三类场景：(a) 仅空间异构 → 一次离线特化； (b) 空间 + 未知时间异构 → 运行时重规划 + 特化 + 切换； (c) 空间 + 可预测时间异构 → 预生成多套计划，运行时按批选择切换。

### 2.2 分层异构分片标注（§4）

- **基础层（沿用 SPMD）**：每个张量两个属性——**DG**（Device Group，张量所在设备的有序列表）与 **DS**（Distributed States，有序字典，key 为逻辑分布式维度 dim、value 为分片数）。三种语义：**Split(dim≥0)** 沿物理维均匀切分、**Duplicate(dim=−1)** 完全复制、**Partial(dim=−2)** 部分持有（需后续规约）。
- **两层扩展（核心创新）**：
  - **DG Union / DS Union**：把单一 DG、DS 泛化为多个设备子集划分与多套 DS 配置，一一对应，形成若干**sharding subgroup（分片子组）**；子组内分片称 **bottom-tier sharding**。
  - **HDim（Heterogeneous Dimension）/ HSize（Heterogeneous Size）**：HSize 为子组数量，HDim 描述**跨子组**的切分方式（如 HDim=0 表示按第 0 维在子组间切分，HDim=−1 表示子组间全复制），称 **top-tier sharding**。
  - 约束：各子组必须是互斥的设备子集；分片的具体形状在运行时决定，与静态标注解耦。
- **设计取舍论证**：
  - **为何两层而非多层**：HDim/HSize 已足以表达复杂非对称分片；两层正好对应"节点/节点组内部 SPMD 负载均衡、节点组之间异构"的现实拓扑；更多层只增大规划与推导空间而无额外表达力。
  - **为何不能只做数据不均匀切分**：对时间异构无效（GPU 故障时必须重配 TP/PP 度）；对空间异构次优（低跨集群带宽下异构 DP 需交换完整梯度，而异构 PP 只传单层激活，通信量小得多）。

### 2.3 分层通信解析（§5）

- 目标：把任意"源标注 → 目的标注"的 Reshard 解析为可执行通信算子，**优先复用集合通信**（CCL 已高度优化，且与多级互联拓扑 Multi-rail/Clos 对齐），不可行时退化为 **BSR**。
- **Bottom-tier 通信**（源/目的 HSize 与 HDim 相同，子组内独立处理）：
  - DS 相同、DG 变 → 各设备 **Identity** 或 **send-receive (SR)** 直传本地分片。
  - DS 不同、DG 同 → 依 DS 变化选 **AR / RS / AG**（图 8）；条件不满足或 DG 不同 → BSR。
- **Top-tier 通信**（HSize 相同、HDim 不同）：
  - DG Union 相同且 DS Union 相同 → 取所有子组的**最细粒度 slice**，按 HDim 变化做 **SplitAR / SplitRS / SplitAG**（图 9）。
  - DG Union 相同但 DS Union 不同 → **先用 bottom-tier 通信对齐 DS Union**，问题归约为上一种情形，再做 top-tier 通信（图 10）。
  - DG Union 不同或 HSize 不同 → BSR。
- **BSR（Batched-Send-Receive，§5.3）**：
  1. 求最细粒度 slice，构造 **BSR table**（每个 slice → 拥有它的设备集合、需要它的设备集合）；
  2. 顺序扫描并套用启发式生成 **BSR scheme**：已拥有则本地拷贝；优先高带宽链路（如 NVLink 优先于 IB）；带宽相同时均衡累计发送负载。
  - 理论说明：最优方案等价于 **广义分配问题（GAP，NP-hard）**；启发式把复杂度降为 **O(pq)**（p 为表项数、q 为单表项最大接收者数），且 p 由切分粒度决定、不随集群规模增长（扩展通常靠 DP 维复制），故大规模下仍可解。
  - 局限：BSR 不处理含 **Partial** 的情形（可扩展为配合 all-reduce 或数值除法，但论文省略）；成本模型只建模 P2P 带宽，不建模路径上单链路瓶颈。

### 2.4 渐进式图特化（§6，应对空间异构）

1. **标注推导（§6.1）**：DG Union 与 HSize 由输入直接继承（语义约束：同一算子所有输入输出必须 DG Union/HSize 一致，否则用户须显式插入 Reshard）；DS Union 在每个子组内独立推导，**退化为标准 SPMD 推导**；HDim 视为"简化的一维分片"按规则推导（如 X 的 HDim=0、W 的 HDim=−1 → 输出 Y 的 HDim=0）。多数算子为恒等传播，仅 Reshard 会改 DG Union/HSize，仅 Dot、Sum、Reshape 等少数算子有自定义规则（表 2）。
2. **算子实例化（§6.2）**：
   - **非本地算子剔除**：DG Union 不含本设备的算子从该设备可执行图中删除（这是形成流水线的前提）；
   - **Reshard 实例化**：bottom-tier 由各子组各自决定（同一 Reshard 在不同设备可分别为 RS 与 BSR）；top-tier 由 DG Union 内所有设备统一实例化（如 SplitAG）。
3. **执行（§6.3 + 附录 A.2）**：可执行图切分为 **Pre / Forward / Backward / Post** 四类子图，作为调度单元；支持 GPipe、1F1B 等调度，各流水线可独立处理不同微批数量与大小；**流水线构建**通过迭代分析 Reshard 中的
