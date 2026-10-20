---
title: "Hetu v2: A General and Scalable Deep Learning System with Hierarchical and Heterogeneous Single Program Multiple Data Annotations"
title_zh: Hetu v2：基于层次化异构SPMD标注的通用可扩展深度学习系统
authors: "Haoyang Li, Fangcheng Fu, Hao Ge, Sheng Lin, Xuanyu Wang, Jiawen Niu, Yuming Zhou, Xupeng Miao, Bin Cui"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-li-haoyang.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 自动推导分布式训练的张量分片与通信模式
tldr: SPMD范式为分布式深度学习训练提供统一标注抽象，但大规模模型训练存在设备差异与数据不均带来的时空负载异构，违背了SPMD的对称划分假设。作者提出Hetu v2，支持层次化与异构的SPMD标注，自动推导张量分片与通信模式以适配异构负载。该系统在保持编程简洁性的同时提升了异构训练下的通信与计算效率，为分布式训练系统的通信优化提供了方法支撑。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 大规模分布式训练存在设备与数据异构，违背SPMD对称划分假设，限制可扩展性。
method: 提出层次化与异构的SPMD标注，自动推导张量分片与通信模式。
result: 在异构训练负载下保持了编程简洁性并改善了通信与计算效率。
conclusion: 为分布式训练系统的通信模式推导与异构适配提供了通用框架。
---

## Abstract
The Single-Program Multiple-Data (SPMD) paradigm provides a unified abstraction to annotate various parallel dimensions in distributed deep learning (DL) training. With SPMD, users can write training programs from the viewpoint of a single device, and the system will automatically deduce the tensor sharding and communication patterns. However, with the recent development in large-scale DL models, distributed training exhibits spatial and temporal workload heterogeneity, arising from both device disparities (e.g., mixed hardware, failures) and data variations (e.g., uneven sequence lengths). Such heterogeneity violates SPMD’s assumption of symmetric workload partitioning, which restricts its ability to express and optimize heterogeneous parallel strategies effectively. To address this, we propose HSPMD within the Hetu v2 system to achieve general and scalable DL training. HSPMD extends SPMD’s declarative annotations to support asymmetric sharding and composes standard communication primitives for hierarchical communication, all while retaining the simplicity of a single-device programming model. HSPMD handles spatial heterogeneity through progressive graph specialization, enabling device-specific execution logic, and addresses temporal heterogeneity via dynamic graph switching. Evaluations on (a) heterogeneous devices, (b) unstable devices, and (c) mixed-length data scenarios show that HSPMD matches or outperforms specialized systems, providing a flexible and efficient solution for modern distributed DL training. Code is available: https://github.com/PKU-DAIR/Hetu .

---

## 论文详细总结（自动生成）

# Hetu v2（HSPMD）论文总结

## 1. 核心问题与研究动机

- **背景**：SPMD（单程序多数据）是分布式深度学习训练的主流抽象，用户只需从单设备视角编写程序，并通过声明式标注（如 Split、Duplicate、Partial）让系统自动推导张量分片与通信模式（代表系统：GSPMD、Alpa、DTensor、Megatron、DeepSpeed）。
- **核心矛盾**：SPMD 的声明式原语建立在两个对称性假设之上：
  - 硬件资源**同构且稳定**；
  - 所有输入数据的训练负载**等价**。
- **现实挑战**：大规模训练中异构性普遍存在，论文归纳为两个本质维度：
  - **空间异构**（某一时刻的负载不均衡）：混合代际 GPU（如 H800+H20）、GPU/节点故障后剩余设备、不同序列长度数据并存；
  - **时间异构**（负载随时间变化）：设备可用性动态变化、批次间序列长度分布漂移。
- **现有方案不足**：
  - MPMD（Pathways、Ray）需为每个设备生成独立程序，编译开销大、可扩展性差；
  - “SPMD + 专用调度器”方案（HexiScale、Oobleck、HotSPa）虽保持单程序模型，但调度器与特定场景紧耦合、需绕过 SPMD 对称性、策略空间受限，难以通用。
- **论文主张**：与其在调度器层做场景定制，不如**下沉到声明式标注原语层**，让 SPMD 自身原生表达非对称性，从而以统一底座覆盖多种异构场景。

## 2. 方法论：HSPMD 系统设计

### 2.1 核心思想
在 Hetu v2 中提出 **HSPMD（Hierarchical and Heterogeneous SPMD）**：通过“层次化 + 异构”的分片标注扩展，把非对称分片与非对称通信变成一等公民；再以“图特化”应对空间异构、“图切换”应对时间异构。两者是场景无关的模块化积木。

### 2.2 分片标注扩展（§4）
- **基础层**：沿用 SPMD 的 Device Group（DG，张量所在设备）与 Distributed States（DS，分片方式），语义包括 Split(dim≥0)、Duplicate(-1)、Partial(-2)，并保留自动推导规则。
- **扩展层（两层分片）**：
  - **DG Union / DS Union**：将单一 DG/DS 泛化为多个“分片子组”（sharding subgroup），每个子组内部独立做标准 SPMD 分片（**bottom-tier sharding**）；
  - **HDim / HSize**：HSize 表示子组数量，HDim 描述张量在子组之间如何切分（**top-tier sharding**，本质是简化的一维分片）。
  - 例如可表达：GPU 0,3 与 GPU 5,6 之间做 TP、GPU 1 与 GPU 5,6 之间做 PP、GPU 2,4 做 CP，并在其上叠加 DP。
- **设计取舍论证**：
  - 不做多层：两层已足够表达复杂非对称，且契合“节点内均衡、节点间异构”的实际拓扑；
  - 不做“仅数据不均分”：时间异构下需重配 TP/PP 度，数据重分片无能为力；空间异构下异构 PP 只传单层激活，通信量远小于异构 DP 的全梯度交换。

### 2.3 层次化通信解析（§5）
将 Reshard 的通信实现分类为：
- **Bottom-tier 通信**（子组内，同构高速链路）：
  - DS 相同、DG 变化 → Identity 或 send-receive；
  - DS 不同、DG 相同 → 标准集合通信（AR/RS/AG），条件不满足时退化为 BSR。
- **Top-tier 通信**（子组间，可能异构慢链路）：
  - 仅 HDim 变化 → 先切到最细粒度切片，再做 SplitAR / SplitRS / SplitAG；
  - DS Union 也变化 → 先做 bottom-tier 对齐 DS Union，再做 top-tier 对齐 HDim。
- **BSR（Batched-Send-Receive）兜底**：当 DG Union 不一致、HSize 不同或无法用集合通信时，构造 BSR 表（切片→拥有者/需求者），用启发式生成传输方案（已拥有则本地拷贝、优先高带宽链路、等带宽时均衡累计发送负载）。该问题本质是 NP-hard 的 GAP，启发式将复杂度降为 O(pq)，且 p 不随集群规模增长。
- **设计理由**：优先用集合通信，是因为 CCL 高度优化、且与多级网络拓扑（NVLink/IB、Multi-rail、Clos）天然对齐；Partial 场景不适用 BSR，但实际中 Partial 多为中间结果，可另行处理。

### 2.4 渐进式图特化（§6，应对空间异构）
流程：定义图 + 标注计划 → **标注推导** → **算子实例化** → 流水线执行。
- **标注推导**：多数算子直接传播标注；仅 Reshard 会改变 DG Union/HSize；仅 Dot、Sum、Reshape 等少数算子有自定义 DS Union/HDim 推导规则（如 3D×2D Dot 的例子）。约束是同一算子的所有输入输出必须 DG Union/HSize 一致，否则需显式插入 Reshard。
- **算子实例化**：
  - 非本地算子剔除（DG Union 不含本设备的算子从可执行图中删除），这是支持流水线的前提；
  - 对剩余 Reshard 按 §5 实例化具体通信算子（如 GPU 0 用 RS、GPU 6 用 BSR）。
- **执行**：算子被组织为 Pre / Forward / Backward / Post 四类可执行子图，支持 GPipe、1F1B 等调度；不同流水线可独立处理不同 micro-batch 数量与大小。张量形状在运行时动态确定，与静态图解耦，从而支持变长输入（附录 A.2 的符号形状扩展，如 `S' = S/2` 约束传播）。
- **每设备维护的数据结构**：定义图、标注计划（各进程相同）、设备专属可执行图、算子元数据（含 BSR 表）；仅占每主机 CPU DRAM 数 MiB。

### 2.5 动态图切换（§7，应对时间异构）
- 不做 checkpoint 重载，而是利用设备间高带宽**在线重分片权重**：权重不含 Partial，可直接用 BSR 在旧图与新图之间桥接。
- **优化**：把所有 BSR 表合并为一张 **Fused BSR 表**，使通信负载全局均衡；并把同一 GPU 对之间的多个 send-receive 融合为单个算子，显著降低 kernel 启动开销。
- **开销可控性论证**：策略数量与集群规模无关——不稳定设备场景任一时刻只保留一个策略；混合长度场景策略数只随“最大序列长度划分粒度”增长。

### 2.6 规划器（§8）
- 基于 profiling 的代价模型（按算子×硬件资源建模），用 ILP / MINLP / 动态规划（PuLP、Pyomo）生成标注计划；
- 离线预生成 + 在线选择/重生成；规划器被设计为可替换的外部模块；
- 保底性质：极端情况下回退到完全同构策略，此时 HSPMD 退化为标准 SPMD。

## 3. 实验设计

- **三大场景**：
  - (a) 异构设备（静态空间异构）；
  - (b) 不稳定设备（空间 + 未知时间异构，含 GPU/节点故障轨迹）；
  - (c) 混合长度数据（空间 + 可预测时间异构）。
- **数据集/负载**：Llama 架构 32B、70B 模型；场景 (c) 使用 CommonCrawl 与 GitHub 数据集，上下文长度 16K / 32K，batch 200K tokens；默认上下文 4K、全局 batch 64。
- **Benchmark 与对比方法**：
  - 标准 SPMD：(i) DeepSpeed（DP+ZeRO、SP）、(ii) Megatron（DP+ZeRO-1、TP、PP、CP）；
  - 场景专用 SPMD 变体：(iii) HexiScale（异构流水线调度器）、(iv) Oobleck（弹性流水线模板）、(v) HotSPa（热切换调度器）。
  - 所有基线均经调优至其最优配置（附录 C 给出各配置下的最优并行策略表）。
- **评价指标**：每步训练时间（含重配置开销）、重配置开销分解、通信量分布、loss 收敛曲线。

## 4. 资源与算力

- 论文明确给出测试床：**16 张 H800 + 32 张 H20 GPU**。
  - H800：81559 MB 显存，990 TFLOPS BF16，NVLink 400 GB/s；
  - H20：97871 MB 显存，148 TFLOPS BF16，NVLink 900 GB/s。
- 具体实验规模：32B 模型训练 100 步（场景 c）；收敛性对比训练 1000 步（附录 B）；弹性训练沿两条故障轨迹进行。
- **未明确说明**：各实验的具体总训练时长/总 GPU-hours、完整集群的节点数与网络拓扑细节（仅提及 NVLink/IB 混合），也未给出能耗或成本数据。

## 5. 实验数量与充分性

- **实验组数概览**：
  - 场景 (a)：7 种设备/模型配置（同构 H800、同构 H20、H800+H20 的 16/24/32 张 H20 组合、70B 三种配置）× 4 个系统；
  - 场景 (b)：7 种配置（C1–C7，含两次 GPU 故障与一次节点级缩容），对比 4 个系统并标注重配置开销；
  - 场景 (c)：2 数据集 × 2 上下文长度 = 4 组，对比 4 个系统，并附序列长度分布与策略切换示例；
  - 案例分析（§10）：C2 策略部署图、逐 rank 计算/通信时间分解、C1→C2 重配置三阶段开销、三种 BSR 算法对比（含通信量分布表）；
  - 附录：收敛性对比（Megatron vs HSPMD 同构 vs HSPMD 异构）、完整策略表、规划/特化/切换的时间开销表。
- **充分性评价**：
  - 覆盖三个代表性异构场景，且每类都有专用 SOTA 基线，横向对比维度较完整；
  - 消融较到位：BSR 的“无启发式 / 非融合 / 融合”三档对比、重配置三阶段拆解、同构场景作为“工程无偏”对照；
  - **公平性考量**：作者明确指出为公平对比，HSPMD 与 Oobleck 均禁用 ZeRO-1 以保留 DP 冗余（并说明这会带来约 15% 性能损失），基线均调至最优；
  - **局限**：每组实验基本为单一模型族（Llama）、单一硬件组合（H800/H20）、最多 48 卡规模；场景 (c) 仅训练 100 步；未做规划器质量的敏感性/消融实验（论文自认规划器优化留待未来）。

## 6. 主要结论与发现

- 在**同构设备**上，HSPMD 与 Megatron/DeepSpeed 性能相当，说明性能差异不来自工程实现差异。
- 在**异构设备**上，HSPMD 一致优于所有基线（如 32B、16 H800+16 H20：HSPMD 7.10s/step，HexiScale 8.92s，DeepSpeed 16.05s，最高约 2.3× 加速）。
- 在**不稳定设备**上，HSPMD 能利用故障后剩余 GPU（如 31 卡、15 卡场景），而对称 SPMD 系统只能整机丢弃并 checkpoint 重启；Oobleck 虽免重启但受限于固定流水线模板，性能落后。
- 在**混合长度数据**上，HSPMD 通过预生成多套异构策略并按批次最大长度动态切换，显著优于固定同构策略的 DeepSpeed/Megatron 与仅支持同构策略切换的 HotSPa（如 32K CommonCrawl：12.56s vs 19.56/27.94/36.52s）。
- **通信开销分析**：SplitAR、BSR 引入的额外开销很小，计算仍占主导（约 79–82%）；Fused BSR 在总通信量不变的前提下，通过更均衡的流量分布与更高 NVLink 利用率，取得最低切换开销。
- **收敛性**：异构策略（C2）与同构策略（C1）的 loss 曲线几乎一致，证明非对称分片与非对称通信不损害收敛。
- **总体论断**：调度器级方案是粗粒度的、场景专用的“自上而下”栈；原语级扩展是细粒度、可组合的“自下而上”通用底座。

## 7. 优点

- **抽象层次选择精准**：把扩展点放在声明式标注原语而非调度器，既保留单设备编程模型的简洁性，又原生表达非对称，避免了为每个场景重造调度器。
- **统一性**：用“空间/时间异构”两个本质维度替代按场景分类，图特化与图切换作为正交积木可组合出三类场景方案，具备良好的可推广性。
- **通信设计贴合硬件**：优先复用高度优化的集合通信、按 bottom/top-tier 匹配多级网络拓扑（NVLink 快、IB 慢），BSR 作为兜底且复杂度可控（O(pq)，p 不随规模增长）。
- **重配置高效**：免 checkpoint 的在线权重重分片 + Fused BSR 融合，避免冷启动；且策略数量与集群规模解耦的论证有说服力。
- **工程完整度高**：原型 87.9K 行 C++ + 16.9K 行 CUDA + 12.5K 行 Python，支持 DP/TP/PP/CP、ZeRO-1、offload、重计算、混合精度；符号形状机制支持变长输入。
- **实验态度审慎**：同构对照排除工程偏差、公平性设置（同禁 ZeRO-1）有明确交代、收敛性验证补齐了“异构不影响正确性”的关键证据、开源代码可复现。

## 8. 不足与局限

-

## 8. 不足与局限

- **实验规模与硬件覆盖面有限**：测试床仅 16×H800 + 32×H20（最多 48 卡），未验证百卡/千卡规模下层次化通信解析、BSR 表构造与图切换开销是否仍可接受；异构组合也只覆盖两代 NVIDIA GPU，未涉及跨厂商加速器、CPU/GPU 混合或异构互联带宽差异更大的情形。
- **模型与负载单一**：实验全部基于 Llama 稠密架构，未覆盖 MoE（专家并行本身带来天然非均衡负载）、多模态、长序列注意力变体等更复杂的非对称场景；场景 (c) 仅使用 CommonCrawl 与 GitHub 两个数据集，工作负载代表性有限。
- **时间异构的验证偏弱**：场景 (b) 仅沿两条预设故障轨迹（两次 GPU 故障 + 一次节点级缩容）评估，未讨论故障频率、故障到达分布、连续抖动等对图切换频率与稳定性的影响；场景 (c) 仅训练 100 步，未给出长时间运行下策略切换的累积开销与收敛稳定性证据。
- **规划器质量缺乏评估**：论文将规划器设计为可替换外部模块，但未报告 ILP/MINLP 的求解时间、搜索空间规模、解的质量与最优性差距，也未做规划器消融（如代价模型误差敏感性、profile 开销、在线重生成延迟）；“极端情况回退同构策略”虽保证可用性，但可能显著浪费异构资源，缺少该退化路径的量化分析。
- **表达力存在明确上限**：两层分片（bottom-tier 标准 SPMD + top-tier 一维 HDim）无法表达更细粒度的非对称（如子组内再异构、二维以上子组拓扑）；同一算子要求所有输入输出 DG Union/HSize 一致，否则需显式插入 Reshard，这可能引入论文未量化的额外通信；HDim 仅支持简化一维切分，对复杂非对称 PP/TP 组合的表达仍是近似。
- **BSR 兜底路径的代价与适用范围**：BSR 本质是 NP-hard 的 GAP 问题，启发式虽将复杂度降至 O(pq)，但不保证最优，论文仅在小规模案例中对比三种 BSR 算法；Partial 场景明确不适用 BSR，实际中若 Partial 出现在非中间结果位置，缺少处理方案。
- **缺少与 MPMD 的直接对比**：论文在动机中将 Pathways/Ray 列为对比对象，但实验中未纳入任何 MPMD 基线，因此“原语级扩展优于调度器级方案”的论断主要建立在与 SPMD 变体（HexiScale/Oobleck/HotSPa）的对比上，对 MPMD 路线的相对优势缺乏实证。
- **公平性设置可能引入偏差**：为与 Oobleck 对齐而禁用 ZeRO-1（作者自述约 15% 性能损失），该设置对 HSPMD 与各基线的影响是否对称未充分讨论；此外基线虽“调至最优”，但最优配置表由作者自行搜索，存在调优投入不对等的可能。
- **评价维度偏窄**：主要指标为每步训练时间与重配置开销，缺少 MFU/硬件利用率、显存占用峰值、通信-计算重叠效率、能耗与成本等维度；也未报告编译/特化/规划阶段的绝对时间及其在总训练时长中的占比（仅附录给出开销表）。
- **系统机制细节与鲁棒性**：图切换需暂停训练并在线重分片权重，切换期间的流水线气泡、对 optimizer state 的处理、切换失败的回滚机制未展开；容错仅体现为“利用剩余设备继续训练”，未讨论与 checkpoint、弹性调度器（如 Kubernetes）的协同及故障恢复的端到端时延。
- **理论保证缺失**：异构策略不损害收敛仅有 1000 步经验曲线支撑，缺少对非对称分片下梯度语义等价性、数值稳定性（如不同精度/不同卡型混合）的分析。
- **可复现性仍有缺口**：代码虽开源，但完整集群拓扑、网络配置、数据集预处理与全部超参未完全披露，外部复现 48 卡异构实验的门槛较高。

## 9. 总体评价与启示

- **定位**：这是一篇“抽象层创新”型系统论文，贡献不在于某个单点性能数字，而在于把异构适配从“调度器定制”下沉为“声明式原语扩展”，并用图特化/图切换两个正交机制统一覆盖空间与时间异构，方法论上具有较好的可组合性与可推广性。
- **证据强度**：三类场景均有专用 SOTA 基线且做同构对照，消融（BSR 三档、重配置三阶段拆解）较扎实，收敛性验证补齐了正确性证据；但规模、模型族与规划器评估构成主要证据缺口，结论更宜表述为“在中等规模、Llama 类负载、两代 NVIDIA GPU 上的有效性验证”，而非普适性结论。
- **可借鉴点**：(1) 以“空间/时间异构”替代场景枚举的问题拆解方式；(2) 优先复用集合通信、按网络层次匹配通信原语、以 BSR 兜底的设计哲学；(3) 免 checkpoint 的在线权重重分片 + Fused BSR 降低切换开销的思路；(4) 策略数量与集群规模解耦的论证方式；(5) 同构退化保底以保证工程可用性的设计。
- **后续方向**：扩展到 MoE/多模态与更大规模集群；评估规划器的求解质量与在线开销；补充 MPMD 对比与更真实的故障/负载轨迹；探索两层以上分片或更一般的非对称表达；完善切换鲁棒性与理论分析。

（完）
