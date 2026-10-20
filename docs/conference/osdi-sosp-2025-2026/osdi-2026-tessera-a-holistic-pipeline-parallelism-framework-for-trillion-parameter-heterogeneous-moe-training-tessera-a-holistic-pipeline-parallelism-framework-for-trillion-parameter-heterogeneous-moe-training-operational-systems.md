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
score: 6.0
evidence: 万亿参数MoE训练流水线并行中的通信与计算重叠
tldr: 针对Qwen等模型由均匀Transformer块转向混合MoE与多种注意力的异构结构、使现有流水线系统在通信与计算重叠后分区失衡的问题，Tessera提出整体性流水线并行框架，将分区选择与通信重叠联合优化。系统还应对万卡级规模下MoE路由波动带来的运行时停顿。该工作面向超大规模分布式训练的通信优化，对数据中心训练负载通信有直接意义。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-001.webp\", \"caption\": \"\", \"page\": 2, \"index\": 1, \"width\": 5926, \"height\": 2736}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-002.webp\", \"caption\": \"\", \"page\": 3, \"index\": 2, \"width\": 4244, \"height\": 3960}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-003.webp\", \"caption\": \"\", \"page\": 4, \"index\": 3, \"width\": 4730, \"height\": 2887}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-004.webp\", \"caption\": \"\", \"page\": 5, \"index\": 4, \"width\": 2992, \"height\": 1984}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-005.webp\", \"caption\": \"\", \"page\": 5, \"index\": 5, \"width\": 6989, \"height\": 3391}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-006.webp\", \"caption\": \"\", \"page\": 11, \"index\": 6, \"width\": 2911, \"height\": 993}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-tessera-a-holistic-pipeline-parallelism-framework-for-trillion-parameter-heterogeneous-moe-training/fig-007.webp\", \"caption\": \"\", \"page\": 13, \"index\": 7, \"width\": 850, \"height\": 320}]"
motivation: 万亿参数MoE模型结构异构，现有流水线系统在通信与计算重叠后分区失衡，且万卡规模下路由波动引发停顿。
method: Tessera提出整体流水线并行框架，联合考虑层代价与通信重叠进行分区，并缓解MoE路由运行时停顿。
result: 框架在异构MoE训练中改善了负载均衡与通信重叠效率。
conclusion: 表明超大规模分布式训练需将通信优化与并行划分协同设计。
---

## Abstract
Training trillion-parameter models relies on pipeline parallelism to scale across thousands of accelerators. However, architectures exemplified by the Qwen family are evolving from uniform Transformer blocks into heterogeneous compositions that combine sparse Mixture-of-Experts (MoE) with different attention variants. This heterogeneity breaks the uniformity assumption behind existing pipeline systems: partitions chosen from serial layer costs become imbalanced after communication is overlapped with computation, because different layer combinations hide different fractions of their communication. Furthermore, at the scale of 10K+ GPUs, MoE routing variation adds a runtime source of stalls, creating transient idle slots that static plans cannot anticipate. We present Tessera, a framework that co-optimizes partitioning and overlap scheduling for heterogeneous pipeline training, and adapts to runtime dynamics. Tessera introduces: (1) an overlap scheduler that synthesizes fine-grained interleavings for each layer combination; (2) an overlap-aware partitioner that uses profiled post-overlap cost to select partitions balanced for parallel execution; and (3) a dynamic bubble optimizer that fills routing-induced idle slots with movable tasks at runtime. Deployed on production clusters for pre-training Qwen3 and Qwen3-Next, Tessera improves throughput by 20%–33% over our production baseline across five workloads at scales from 4,096 to 12,288 GPUs, reaching 39% MFU on a trillion-parameter model. In controlled experiments against Megatron-Core MoE with public recipes, Tessera achieves up to 1.24× higher MFU.

---

## 论文详细总结（自动生成）

# Tessera 论文中文结构化总结

## 1. 核心问题与研究动机

- **背景**：万亿参数模型训练依赖流水线并行（PP）扩展到数千至上万加速器。以 Qwen 家族为代表的架构正从“均匀 Transformer 块”演化为**异构组合**：稀疏 MoE + 多种注意力变体（如 Qwen3-Next 每 4 层按 3:1 交错 3 个 Gated DeltaNet 线性注意力层与 1 个标准 softmax 注意力层，每层后接稀疏 MoE）。
- **核心问题一：重叠后分区失衡**。现有 PP 系统按“串行层代价”分区，隐含假设各阶段通信隐藏比例（overlap efficiency）一致。异构模型下该假设失效：实测 chunk 对 C-C 重叠可降延迟 41.6%，D-D 仅 14.0%，重叠收益差异达 3×；串行均衡分区在重叠后瓶颈成本反而高 1.14×。
- **核心问题二：分区与重叠调度的循环依赖**。分区决定哪些层组合共同执行、从而决定可形成的 overlap pair；但要选好分区又需先调度并测量每个组合的 post-overlap cost——二者互为前提，形成静态规划环。
- **核心问题三：运行时随机性**。万卡以上规模下 MoE 路由波动（Qwen3-Next 为 TopK=10/512 的极稀疏路由，单专家覆盖更低、方差更大）造成瞬时空闲槽，静态计划无法预测。
- **整体含义**：现有“先按串行代价分区→套固定重叠模板→静态执行”的瀑布式流水线在异构 MoE 下三步全部失效；领域专家需为每个新架构手工调参数周。论文提出 Tessera，将分区与重叠调度联合优化，并在运行时吸收路由抖动。

## 2. 方法论

### 2.1 总体思想
- 固定高层流水线调度模板（如 interleaved 1F1B），在其内部**联合优化分区与重叠调度**；采用“先 profile、后分区”的顺序打破循环依赖。
- 静态规划器产出执行计划，与计划无关的执行引擎在运行时解释执行；动态气泡优化器（DBO）在不改变分区边界与 backbone 顺序的前提下填补运行时空闲槽。
- 作为宿主训练框架（Megatron-LM）的插件，无需侵入式改造训练循环。

### 2.2 静态规划三步流程

**Step 1：重叠图与候选空间**
- 依模板构建重叠图 G=(S,E)：节点为虚拟流水线阶段，边表示同一 rank 上可能并发执行的重叠机会；边记录两端操作的方向（如 forward-on-s 与 backward-on-t），同一端点对可产生多条边；自环表示同一虚拟阶段不同 microbatch 间的重叠。
- 以串行代价均衡的基线分区为起点，为每个阶段生成**有界候选集**（在基线边界附近扰动得到的连续层范围），以控制搜索规模。

**Step 2：重叠调度与物理 Profiling**
- 将每条边上的候选对分解为两个 **task DAG**（每个 chunk 一个）；task 为原子调度单元（如 Dispatch、MLP、Combine、Wgrad），带 duration、资源类型（Comp/Comm）与依赖约束。
- 用**事件驱动 list 调度器**（Algorithm 1）在 Comp/Comm 两类资源上最小化 pair makespan，含两条启发式：
  - **Backbone-first alignment**：backbone（关键路径）任务优先，gap-fit 规则选择时长最接近互补资源剩余窗口的任务；
  - **Conditional deferral**：movable 任务仅在“适配当前 gap 且不延长 makespan”时放置，否则推迟，之后按 best-fit 回填残余槽或置于尾部。
- 在**参考设备组**（与目标 rank 同 TP/EP 拓扑的专用设备集）上实测 **ground-truth post-overlap cost**；按 device-mesh class 与 chunk 规格缓存，相同规格的边共享 profile，避免重复测量。

**Step 3：重叠感知的分区选择（MILP）**
- 目标：最小化最大被选边的 post-overlap 成本 T_dom（作为稳态瓶颈的代理目标）。
- 约束：每个节点恰选一个候选；边变量 z_{e,c,d} 与两端节点选择一致；T_dom ≥ 每条边所选对的 profiled 成本；另加连续非重叠层范围、设备拓扑兼容、每 rank 内存容量约束。
- 复杂度由 rank 级枚举的 O(N·K^{vp}) 降为 O(N·vp·K²)，瓶颈约束数为 |E|。
- **Profile 引导剪枝**：丢弃 profiled 成本超过基线 T_base 的候选对（不可能改进），并将失去全部有效对的候选剔除，问题规模缩小约 3–5×，且保留所有 T_dom ≤ T_base 的解。
- **计划合成**：将选中标签物化为最终执行计划，每条边替换为 Step 2 合成的细粒度重叠调度。

### 2.3 动态气泡优化（DBO）
- **气泡预测**：复用 MoE dispatch 前已可得的 routing metadata，在 EP 组内以轻量 collective 交换每专家 token 计数（标量而非张量，无全集群同步）；结合离线 profile 估计各 rank 负载与空闲槽位置/大小。
- **槽位预标注**：静态规划时在 MoE 计算
