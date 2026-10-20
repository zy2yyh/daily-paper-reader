---
title: "{DynaRL}: Flexible and Dynamic Scheduling of {Large-Scale} Reinforcement Learning Training"
title_zh: DynaRL：大规模强化学习训练中的灵活动态调度
authors: "Yuanqing Wang, Hao Lin, Junhao Hu, Chunyang Zhu, Quanlu Zhang, Zhen Guo, Yuchen Zhang, Xu Fu, Si Xu, Bo Dai, Zixiao Huang, Chao Yu, Boxun Li, Guohao Dai, Zhi Yang, Yu Wang"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-wang-yuanqing.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: 大规模强化学习训练中计算、内存与通信资源的动态重分配
tldr: 针对大规模强化学习训练中重尾 rollout、多轮工具交互和时变瓶颈导致静态资源分配大量闲置的问题，DynaRL 将整个训练流水线建模为动态超图，并统一迁移计算、内存与通信资源。系统据此持续感知负载变化并动态重分配异构组件资源。实验表明该调度方法能减少空闲算力、缩短训练时间，为动态分布式训练资源管理提供了通用控制面。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-dynarl-flexible-and-dynamic-scheduling-of-large-scale-reinforcement-learning-training/fig-001.webp\", \"caption\": \"\", \"page\": 6, \"index\": 1, \"width\": 640, \"height\": 436}]"
motivation: 大规模强化学习训练因重尾 rollout 和时变瓶颈呈现强动态性，静态资源分配会造成大量算力闲置。
method: DynaRL 用动态超图建模完整 RL 流水线，并通过统一资源迁移接口动态重分配计算、内存与通信资源。
result: 该调度方法能随负载变化调整异构组件资源，减少空闲算力并缩短训练时间。
conclusion: 结果表明持续演化的资源控制面可有效应对大规模强化学习训练的动态性。
---

## Abstract
Modern reinforcement learning (RL) workloads, powering large language models, long‑horizon reasoning, and agentic systems, exhibit extreme dynamicity due to heavy‑tailed rollouts, irregular multi‑turn tool interactions, and time‑varying bottlenecks. Static resource allocations in today’s distributed RL systems leave large fractions of compute idle and prolong training. This paper presents DynaRL, the first RL system that dynamically reallocates computation, memory, and communication resources across heterogeneous RL components. DynaRL models the entire RL pipeline with a dynamic hypergraph that serves as a centralized, continuously-evolving control surface. Supported by a unified resource migration interface and context‑aware data routing, the scheduler reallocates GPUs from overprovisioned components to the current bottleneck via a combination of a multi-level scheduling algorithm and fine-grained resource migration. Comprehensive evaluation demonstrates that DynaRL improves end-to-end throughput on math-reasoning and agentic RL workloads by up to 1.98×, with negligible online scheduling overhead.

---

## 论文详细总结（自动生成）

# DynaRL 论文中文总结

## 1. 核心问题与整体含义（研究动机与背景）

- **背景**：强化学习（RL）已从传统控制/游戏任务扩展到支撑大语言模型、长程推理与 Agentic 系统，训练规模达到数百 GPU 的分布式集群。现代 RL 流水线由异构组件构成（rollout 生成、推理、多步工具调用、环境交互、策略优化训练），组件间以高度不规则、时变的方式交互。
- **核心问题**：RL 工作负载存在极端动态性，主要来自两方面：
  - **重尾生成（long-tail generation）**：自回归推理的响应长度呈偏态分布（论文刻画中 decode 时间均值 62.85s、最大 259.51s，约 95% 查询在 150s 内完成），长尾请求拖住整个 rollout 阶段，可浪费高达 60% 的可用算力。
  - **不规则外部交互**：Agentic RL 中模型反复调用搜索引擎、Python 解释器、定理证明器等工具，工具调用时间高度偏斜（均值 1.91s、最大 30.05s）；推理引擎默认 FIFO 调度导致 KV cache 抖动与反复 re-prefill。
- **现状缺陷**：现有 RL 系统（verl、RLinf 等）基本采用**静态资源划分**，预先给各组件分配 GPU 并全程保持不变，与负载的时变需求根本性错配，造成吞吐不稳定、资源低效、训练时间延长。
- **论文主张**：动态调度应成为下一代分布式 RL 系统的**一等设计原则**；系统需能观测运行时信号、检测瓶颈迁移、在保证依赖与训练状态正确的前提下实时重分配资源。
- **整体含义**：DynaRL 是首个在异构 RL 组件间动态重分配计算、内存与通信资源的 RL 系统，将整条流水线转化为单一、持续演化的可控运行时对象。

## 2. 方法论

### 2.1 核心思想
- 把整个 RL 流水线建模为**动态超图（dynamic hypergraph）**，作为调度器与运行时之间的集中式、持续更新控制面；配合**统一资源迁移接口**与**上下文感知数据路由**，实现预测性、低开销、组件无关的动态调度。
- 以**资源过度供给（overprovision）**为主要调度信号：当某组件增加资源仅带来边际吞吐提升、且削减少量资源不明显降低性能时，判定其过度供给，回收资源并迁移给当前瓶颈组件。

### 2.2 关键技术细节

- **动态超图抽象**
  - `HyperNode` 表示一个逻辑组件的 WorkerGroup：静态属性含组件类型（动态/静态）、可中断性（always/regular/never）、前驱/后继；动态状态含同构节点列表、已分配资源、利用率。
  - `HyperEdge` 表示组件间数据流，携带 affinity（偏好节点）与 progress（数据处理进度），用于跟踪依赖就绪度与反压。
  - 通过 Python `ast` 静态分析 + 运行时 hooks 自动构建超图，无需手工标注。

- **统一资源迁移接口（interrupt–schedule–migrate 模式）**
  - 流程：WorkerGroup 管理器在可中断点调用 `interrupt()` → 调度器判断是否需重平衡并计算新分配 → 执行 `migrate(dest_resources)` → 管理器按组件特定策略迁移。
  - 三种迁移策略：
    - **RebootMigration**：关闭旧 Worker、保存状态、在目标资源重启；通用但开销较高，适合轻量组件，也用于异常（挂起/死循环）时的强制恢复。
    - **WorkloadMigration**：暂停待释放资源上的 Worker，收集并重分发在途/待处理数据给保留 Worker；适用于无状态组件（SGLang、vLLM 等推理引擎）。
    - **p2pMigration**：先在目标资源启动新 Worker 并保持旧 Worker 活跃，通过规划的点对点路径传输模型参数与优化器状态，遵循 create-before-destroy，用分布式 barrier 保证原子性；失败可回滚。适用于 Megatron-LM 等有状态训练器，多数情况约 1 秒完成。
  - 每个 Worker 运行后台心跳守护进程接收调度命令。

- **上下文感知数据路由**
  - 每个 WorkerGroup 内的本地 `DataRouter` 为数据附加：affinity（上次处理位置及偏好/必须/无需同 Worker）与 status（如序列 ID、已完成步数）。
  - `distribute_data` 拦截发送队列按 affinity 路由；接收端 `priority_schedule` 决定执行顺序，实现细粒度控制。

- **多级动态调度算法**
  - 形式化目标：设静态组件吞吐近似线性 `T_S(R_S) ≈ α_S R_S`，动态组件为凹函数且存在过度供给点 `R^oa_D`；目标为
    `max_R min_{Ci∈C} T_i(R_i)  s.t.  Σ R_i ≤ R_total`（同步流水线吞吐取各阶段最小值）。
  - 调度分三级：**WorkerGroup 级**（组件间资源分配）、**Worker 级**（组内负载均衡与扩缩容）、**数据级**（请求分发与优先级）。
  - **Phase 1 过度供给检测**：对动态组件计算 `U_D(t) = (1/R_D) Σ_j KV_used_j / KV_capacity_j`；若超过 θ（如 80%）的引擎利用率低于 `U_low` 并持续 T 秒，则标记为候选让出者。`U_low` 可取历史利用率的第 p 百分位（如 20th）。
  - **Phase 2 重分配规划**：① 生成 K 个离散缩减候选；② 用轻量统计回归模型（预训练 profiling + 在线回归校准）预测缩减后吞吐；③ 评估释放资源再分配后的全局吞吐 `τ(k)`；④ 选 `k* = argmax τ(k)`。
  - 复杂度 `O(K × |C|)`（K 通常 3–10，|C| 通常 3–7），与 GPU 总数无关；128 GPU 下调度可在 200ms 内完成。

- **请求数据调度（优先级感知）**
  - 观察到：第 0 轮响应显著更长，后续轮次生成长度单调递减；早轮计算重、晚轮短而便宜。
  - 优先级键 `(k_r, m_r)`：`k_r` 为已完成工具调用次数，`m_r` 为前缀树 KV cache 中的前缀长度（同轮内 tie-breaker）；按此排序选 top-B 组成批次。
  - 收益：提高 KV cache 复用概率、避免 re-prefill；加速临近完成的 episode，降低平均完成时间；在晚轮不短的工作负载下可退化为近 FIFO。

## 3. 实验设计

- **数据集 / 场景**
  - **数学推理 RL**：AReaL-boba-Data（整合 DeepScaleR、Open-Reasoner-Zero、Light-R1、DAPO、NuminaMath 的 AoPS/Olympiad 子集、ZebraLogic），过滤过简问题。
  - **多轮 Agentic RL**：rstar2-agent（含 DAPO 训练集任务、OpenMathReasoning 的 AoPS 论坛问题、Project Euler）。
- **模型规模**：1.5B、7B、32B 参数模型。
- **对比方法（Benchmark）**
  - 数学推理：verl v0.5、RLinf、RLHFuse（无公开 artifact，作者在 RLinf 内用相同 rollout 引擎与训练后端复现其核心融合逻辑）。
  - Agentic RL：仅对比 RLinf（verl 在该实验只支持 FSDP，明显慢于 Megatron-LM 后端）。
  - 统一使用 SGLang 作 rollout 引擎、Megatron-LM 作训练后端。
- **评价指标**：RLHF 吞吐（tokens/sec，全局 batch 总 token 数除以一次迭代时间）；Agentic RL 用每秒处理请求数（requests/sec）；均在 warm-up 后取 10 个训练迭代平均。
- **配置**：rollout batch size 512、group size 16、序列长度 28,672；1.5B/7B/32B 的训练 TP 分别为 2/4/8，rollout TP 为 1/2/4，且扩展 GPU 数时保持不变。

## 4. 资源与算力

- **硬件**：H100 集群，16 节点、共 128 张 NVIDIA H100-80GB；每节点 8 GPU、2 颗 Intel Xeon Platinum 8558@2.1GHz（48 核）、2TB 内存；节点内 NVLink，节点间每节点 8 张 Mellanox ConnectX-7 RDMA NIC（400 Gbps，RoCEv2）。
- **实验规模**：主要在 64 GPU 与 128 GPU 两种规模下评测。
- **训练时长**：论文未明确报告端到端训练总时长或总 GPU 小时数；仅提及指标取 warm-up 后 10 个迭代平均、迁移开销占比 <0.5%~1%、调度决策 <200ms 等。

## 5. 实验数量与充分性

- **实验组数概览**
  - 数学推理端到端：3 种模型规模 × 2 种 GPU 规模 × 4 个系统（verl、RLinf、RLHFuse、DynaRL）。
  - Agentic RL 端到端：3 种模型规模 × 2 种 GPU 规模 × 3 种配置（RLinf、DynaRL、DynaRL+Priority）。
  - 调度时间线剖析（64 GPU、7B 数学推理，静态 / 动态 / 动态+优先级三组）。
  - 迁移开销：trainer 与 rollout 各在 3 模型 × 2 GPU 规模下测量。
  - 参数敏感性：`U_low` 取 5 个值、`T` 取 5 个值（1.5B 数学推理）。
  - 正确性验证：1.5B、64 GPU 下静态 vs 动态的 reward 曲线对比。
- **充分性评估**
  - **较充分之处**：覆盖两种代表性 RL 工作负载、三种模型规模、两种集群规模；包含消融（动态分配 vs 加优先级调度）、开销分析、参数敏感性、收敛性验证，维度较全面。
  - **客观性与公平性**：基线均为 SOTA 开源系统，统一 rollout/训练后端，指标沿用 verl 定义；RLHFuse 因无公开代码由作者复现，存在一定实现差异风险。
  - **不足**：作为"首个"系统，缺少与其他动态/弹性 RL 调度系统的直接对比；Agentic RL 仅与 RLinf 对比；未覆盖 PPO 等更复杂的七组件流水线（背景中提到但实验以 GRPO 类为主）；敏感性分析仅在 1.5B 数学推理上做。

## 6. 主要结论与发现

- **数学推理 RL**：DynaRL 相对 verl/RLinf 取得 **1.27×–1.98×** 吞吐提升；64 GPU 下 1.5B/7B 约 1.43×–1.55×；128 GPU 下 32B 达 1.98×（对 verl）与 1.40×（对 RLinf）；相对 RLHFuse 提升 1.21×–1.42×。32B 在 64 GPU 下 verl 因 OOM 失败。
-
