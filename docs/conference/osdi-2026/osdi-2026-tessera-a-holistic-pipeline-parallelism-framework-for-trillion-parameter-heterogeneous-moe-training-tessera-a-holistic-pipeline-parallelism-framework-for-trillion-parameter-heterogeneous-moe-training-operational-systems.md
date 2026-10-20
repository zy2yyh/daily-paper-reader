---
title: "Tessera: A Holistic Pipeline Parallelism Framework for {Trillion-Parameter} Heterogeneous {MoE} Training (Operational Systems)"
title_zh: Tessera：面向万亿参数异构MoE训练的整体流水线并行框架
authors: "Weifang Hu, Langshi Chen, Man Yuan, Youyang Yao, Xiulong Yuan, Li Tian, Yong Li, Wei Lin, Xuanhua Shi, Zhengping Qian, Jingren Zhou"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-hu-weifang.pdf"
tags: ["query:dcn-comm"]
score: 7.0
evidence: 面向万亿参数分布式训练的通信重叠流水线划分
tldr: 万亿参数MoE模型层结构异构，现有流水线并行按串行层代价划分阶段，在计算与通信重叠后出现负载失衡，且万卡规模下专家路由抖动带来额外停顿。Tessera提出整体流水线并行框架，以重叠后的实际耗时指导分区并调度运行时抖动。实验显示其在异构MoE训练中显著降低流水线气泡并提升吞吐。该工作说明通信感知的分区建模对超大规模训练至关重要。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-001.webp\", \"caption\": \"\", \"page\": 2, \"index\": 1, \"width\": 5926, \"height\": 2736}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-002.webp\", \"caption\": \"\", \"page\": 3, \"index\": 2, \"width\": 4244, \"height\": 3960}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-003.webp\", \"caption\": \"\", \"page\": 4, \"index\": 3, \"width\": 4730, \"height\": 2887}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-004.webp\", \"caption\": \"\", \"page\": 5, \"index\": 4, \"width\": 2992, \"height\": 1984}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-005.webp\", \"caption\": \"\", \"page\": 5, \"index\": 5, \"width\": 6989, \"height\": 3391}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-006.webp\", \"caption\": \"\", \"page\": 11, \"index\": 6, \"width\": 2911, \"height\": 993}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-007.webp\", \"caption\": \"\", \"page\": 13, \"index\": 7, \"width\": 850, \"height\": 320}]"
motivation: 现有流水线并行系统假设层结构均匀，难以应对MoE与多种注意力变体混合的异构模型以及万卡规模下的通信重叠失衡。
method: 提出整体流水线并行框架，按通信与计算重叠后的实际耗时划分流水线阶段，并针对MoE路由抖动进行运行时调度。
result: 在万亿参数异构MoE模型上获得更均衡的分区与更低的流水线气泡，显著提升大规模训练吞吐。
conclusion: 表明面向异构层组合与通信重叠的联合建模是扩展超大规模MoE训练的关键。
---

## Abstract
Training trillion-parameter models relies on pipeline parallelism to scale across thousands of accelerators. However, architectures exemplified by the Qwen family are evolving from uniform Transformer blocks into heterogeneous compositions that combine sparse Mixture-of-Experts (MoE) with different attention variants. This heterogeneity breaks the uniformity assumption behind existing pipeline systems: partitions chosen from serial layer costs become imbalanced after communication is overlapped with computation, because different layer combinations hide different fractions of their communication. Furthermore, at the scale of 10K+ GPUs, MoE routing variation adds a runtime source of stalls, creating transient idle slots that static plans cannot anticipate. We present Tessera, a framework that co-optimizes partitioning and overlap scheduling for heterogeneous pipeline training, and adapts to runtime dynamics. Tessera introduces: (1) an overlap scheduler that synthesizes fine-grained interleavings for each layer combination; (2) an overlap-aware partitioner that uses profiled post-overlap cost to select partitions balanced for parallel execution; and (3) a dynamic bubble optimizer that fills routing-induced idle slots with movable tasks at runtime. Deployed on production clusters for pre-training Qwen3 and Qwen3-Next, Tessera improves throughput by 20%–33% over our production baseline across five workloads at scales from 4,096 to 12,288 GPUs, reaching 39% MFU on a trillion-parameter model. In controlled experiments against Megatron-Core MoE with public recipes, Tessera achieves up to 1.24× higher MFU.

---

## 论文详细总结（自动生成）

# Tessera 论文结构化总结

## 一、核心问题与研究动机

- **背景演进**：以 Qwen 系列为代表的基础模型正从“均质 Transformer 堆叠”转向**异构组合架构**——将稀疏 Mixture-of-Experts（MoE）与多种注意力变体（如 Gated DeltaNet 线性注意力与标准 softmax 注意力按 3:1 交错）混合。Qwen3-Next 即为典型代表。
- **既有流水线系统的失效**：传统流水线并行（PP）系统建立在“统一 Transformer 砖块”假设之上，即各层算力强度与通信模式相近，因此可用“串行层代价均衡 + 固定重叠模板”应对。异构 MoE 打破该假设，暴露出三重叠加问题：
  1. **重叠调度空间组合爆炸**：即便最小 2 阶段划分，也能产生多达 8 种不同的重叠对（overlap pair）类型，每种需要结构不同的细粒度交错方案，固定模板无法覆盖。
  2. **分区目标具误导性**：按串行代价均衡的分区在“通信-计算重叠”后可能严重失衡。实测中不同块对的重叠收益差异达 3×（如 C-C 对延迟降低 41.6%，D-D 对仅 14.0%），串行均衡分区的瓶颈后重叠代价比重叠感知分区高 1.14×。
  3. **运行时随机性**：万卡以上规模下，MoE 路由抖动（每 token 仅激活 512 专家中的 10 个，路由覆盖度低）导致每轮各 rank 负载波动，产生静态计划无法预知的瞬时空闲槽。
- **核心矛盾（循环依赖）**：分区决定哪些层组合共同执行、从而决定可用的重叠对；而评估分区优劣又需先调度并测量每个组合的“后重叠代价”（post-overlap cost）。现有系统靠“假设重叠效率恒定”回避该耦合，实践中需领域专家为每个新架构手工调参数周。
- **整体含义**：论文主张对**分区与重叠调度进行联合优化**，并叠加**运行时自适应**，才能支撑万亿参数异构 MoE 的高效训练。

## 二、方法论（Tessera 框架）

Tessera 作为宿主训练框架（Megatron-LM）的插件，固定高层流水线调度模板（如交错 1F1B），在其内部协同优化分区与重叠，并引入运行时动态优化。整体由**静态规划器**与**动态气泡优化器（DBO）**构成。

### 1. 静态规划三步工作流

- **Step 1：重叠图与候选空间构建**
  - 由调度模板构造重叠图 \(G=(S,E)\)：节点为虚拟流水级，边为同一 rank 上可能并发执行的重叠机会，边记录两个端点操作的**执行方向**（故同一端点对可对应多条边，自环表示同一虚拟级不同微批间的重叠）。
  - 以“串行代价均衡”分区为基线，对每个阶段在其边界邻域内扰动，生成有界候选集 \(C_s\)（每个候选是一段连续层区间）。
  - 后重叠代价被约束在 \([\max(T_c,T_d),\, T_c+T_d]\) 区间内（完美重叠 ~ 零重叠）。

- **Step 2：重叠调度与真实性能剖析**
  - 将每个重叠对分解为两个**任务 DAG**（任务如 Dispatch、MLP、Combine、Attn、MLP-W 等原子调度单元，含时长、资源类型、依赖约束）。
  - 采用**事件驱动列表调度器**（算法 1）在 Comp/Comm 两类资源上最小化 makespan，含两大启发式：
    - **主干优先对齐（backbone-first）**：主干任务（决定阶段时延的关键路径任务）优先，按“时长最接近互补资源剩余窗口”的 gap-fit 规则选择；
    - **条件延迟（conditional deferral）**：可移动任务（如 Wgrad、梯度归约）仅在“不延长当前对 makespan”时放置，否则延迟，后续按最佳适配回填至残余槽或尾部。
  - 在**参考设备组**（与目标 rank 相同 TP/EP 拓扑的专用设备集）上实际执行合成调度，测得**真实后重叠代价** \(T_{e,c,d}\)。按“设备网格类别 + 块规格”缓存，相同规格的剖析结果可跨 rank/副本复用，每个不同重叠对仅剖析一次。

- **Step 3：重叠感知分区选择（MILP）**
  - 将分区选择建模为图标注问题：二值变量 \(y_{s,c}\) 选阶段候选、\(z_{e,c,d}\) 激活边上的候选对。
  - 目标：最小化瓶颈后重叠边代价 \(T_{dom}\)，约束包括：每节点恰选一个候选、边变量与端点选择一致、\(T_{dom} \ge \sum_{c,d} T_{e,c,d} z_{e,c,d}\)，另加连续不重叠层区间、设备拓扑兼容、每 rank 内存容量等约束。
  - **复杂度**：解耦为阶段级变量 + 相邻一致性约束，二值变量数从 rank 级枚举的 \(O(N\cdot K^{v_p})\) 降至 \(O(N\cdot v_p\cdot K^2)\)。
  - **profile 引导剪枝**：以基线分区瓶颈代价 \(T_{base}\) 为上界，丢弃代价超过 \(T_{base}\) 的候选对；若某候选在所有关联边上均无有效对则移除。实践中问题规模缩减 3–5×，所有配置在 5 秒内求解。
  - **计划合成**：将选中标签实例化为最终执行计划，每条边替换为 Step 2 合成的细粒度重叠调度。

### 2. 动态气泡优化（DBO）

- **气泡预测与槽位定尺**
  - 复用专家分发前即可获得的**路由元数据**：EP 组内通过搭载在 MoE dispatch 路径上的轻量集合通信交换“每专家 token 计数”（仅标量、组内范围，无全局同步）。
  - 静态规划阶段**预标注目标槽位**（位于 MoE 计算任务之后的结构边界，因两个块来自不同微批、token 分布独立，槽位大小逐轮波动）。
  - 利用观测到的 token 分布与离线剖析的静态调度，**异步**估算每个目标槽的可填充空闲窗口 \(b_\beta\)，与有用计算重叠进行，非正窗口跳过。

- **可移动任务注入**
  - 维护**可移动任务池**（如 Wgrad，在依赖就绪后入池，硬截止时间为迭代结束），按截止紧迫度组织优先队列；池受每 GPU 容量与内存余量约束，超限则保持原位置不延迟。
  - **贪心注入策略**（算法 2）：在预标注槽位前异步触发，从池中挑选可行且得分 \(S(\sigma,b_\beta)\) 最高者（兼顾时长贴合与截止松弛小）；无合适任务则留空。
  - **保证执行**：若某任务截止松弛趋近 0 仍无槽位，则抢占式注入主流，正确性优先于优化。
  - 槽位定尺与任务选择总开销 < 10 μs。

### 3. 系统实现要点

- 约 11,000 行 Python + 2,000 行 C++，集成 Megatron-LM。
- **计划无关执行引擎**（C++）：以无锁有限状态机协调前/反向 Torch 线程在 yield 点同步，后台线程处理可移动任务，将静态计划作为“规格”解释执行，避免硬编码控制流。
- **任务边界定制**：通过 `advance(TaskName)` 轻量探针 API 切分任务（不移动张量计算），`register_task()` 注册可移动任务；支持 YAML 自定义交错策略。
- **微批填充（padding）**：注入“影子动作”（跳过计算但发送伪张量的空操作）解耦“微批数须整除 PP 规模”的约束，不计入损失/梯度/优化器，开销约 0.5%。

## 三、实验设计

- **场景与模型**
  - **生产环境**：阿里巴巴生产集群，NVIDIA Hopper GPU 服务器（每台 8 卡），RoCE 高速网络；覆盖 Qwen3-L、Qwen3-XL（万亿）、Qwen3-Next-M、Qwen3-Next-L（万亿）、Qwen3-Next-XL（万亿）五个工作负载，规模 4,096–12,288 GPU。
  - **受控对比**：256 GPU 集群上评估 Qwen3-235B、DeepSeek-V3（MLA + MTP + 稀疏 MoE）、Nemotron-3 Super（混合 Mamba/Attention + 潜在 MoE + MTP）。
  - 数据：内部专有数据集与开源数据集混合。
- **基准与对比方法**
  - **内部优化基线**：阿里内部 Megatron-LM 分支（已含交错 1F1B、原生通信重叠、生产调优的 FLOPs 均衡分区），用于隔离 Tessera 的增量收益。
  - **Megatron-Core MoE v0.16.1**：采用 Megatron-MoE-ModelZoo 与 Megatron-Bridge 的公开 per-model recipe（含 EP 通信重叠），用于可复现的受控对比。
- **评价指标**：MFU、端到端吞吐、规划开销、峰值内存。
- **实验类型**
  - 端到端生产性能（表 2）+ 两周“热升级”分阶段追踪（图 9，4 月 9 日部署静态规划器、4 月 16 日启用 DBO）。
  - 迭代时间分解案例研究（图 10，两个 Qwen3-Next 工作负载）。
  - 与 SOTA 对比（图 11，三种开源 MoE 模型）。
  - **消融 1：分区与重叠协同优化**（图 12，Partition A 串行均衡 vs Partition B 重叠感知，各含/不含重叠调度）。
  - **消融 2：重叠调度器对比**（图 13，FIFO / +Align / +Defer / ILP，EP8 与 EP32 两种宽度、理论 vs 实测 CDF，536 与 690 个调度实例）。
  - **消融 3：DBO**（表 3，Baseline 与 Overlap-Aware 两种静态计划 × static / w-DBO / always-keep 三配置，含峰值内存）。
  - **规划开销**（表 4，4 种 PP8-C2/C4 配置的 MILP 规模与求解时间；序列长度 4K–128K 的剖析耗时）。
  -
