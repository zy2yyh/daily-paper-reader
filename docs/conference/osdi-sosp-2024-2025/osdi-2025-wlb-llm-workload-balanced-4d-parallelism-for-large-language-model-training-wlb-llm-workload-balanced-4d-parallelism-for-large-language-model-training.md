---
title: "{WLB-LLM}: {Workload-Balanced} 4D Parallelism for Large Language Model Training"
title_zh: WLB-LLM：面向大模型训练的负载均衡四维并行
authors: "Zheng Wang, Anna Cai, Xinfeng Xie, Zaifeng Pan, Yue Guan, Weiwei Chu, Jie Wang, Shikai Li, Jianyu Huang, Chris Cai, Yuchen Hao, Yufei Ding"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-wang-zheng.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: 在微批次间平衡计算与通信负载以优化大模型训练
tldr: 大模型训练在流水并行与上下文并行层面存在明显的负载不均衡，导致计算与通信等待浪费。本文提出WLB-LLM，在流水并行层采用负载感知的变长文档打包平衡微批次间计算与通信量，并在上下文并行层引入细粒度的按文档分片策略。实验表明该方法显著缓解了各worker间的负载差异，提升了训练吞吐与效率，为大规模分布式训练的资源均衡调度提供了实用方案。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
motivation: 大模型训练在流水并行和上下文并行层面存在负载不均，造成计算与通信资源浪费。
method: WLB-LLM采用负载感知的变长文档打包与细粒度按文档分片策略，均衡各worker的计算与通信负载。
result: 实验表明该方法缓解了worker间负载差异，提升了训练吞吐与效率。
conclusion: 该工作为大规模分布式训练中的负载均衡并行调度提供了实用方案。
---

## Abstract
In this work, we present WLB-LLM , a W ork L oad- B alanced 4D Parallelism for L arge L anguage M odel Training. We first thoroughly analyze the workload imbalance issue in LLM training and identify two primary sources of imbalance at the pipeline parallelism and context parallelism levels. Then, to address the imbalance issue, at the pipeline parallelism level, WLB-LLM incorporates a workload-aware variable-length document packing method to balance the computation and communication workload across micro-batches. Additionally, at the context parallelism level, WLB-LLM introduces a novel fine-grained per-document sharding strategy, ensuring each worker within a context parallelism group has an identical workload. Comprehensive experiments under different model scales demonstrate that WLB-LLM significantly mitigates the workload imbalance during 4D parallelism LLM training and achieves an average speedup of 1.23× when applying WLB-LLM in our internal LLM training framework.

---

## 论文详细总结（自动生成）

# WLB-LLM：面向大模型训练的负载均衡四维并行 —— 论文深度总结

## 1. 核心问题与整体含义

- **背景**：LLM 规模与上下文窗口持续增长（如 LLaMA3-405B 使用 16K H100 训练数月），训练算力成本极高，任何端到端效率提升都能转化为可观节省。当前主流训练范式是 **4D 并行**：数据并行（DP）、流水并行（PP）、上下文并行（CP）、张量并行（TP）。
- **核心问题**：大规模训练中大量 GPU 处于**欠利用**状态，根源是**负载不均衡**。作者在内部 8K-GPU、128K 上下文、405B 模型的真实任务中观测到：最慢 GPU 的计算延迟比平均高 **1.44×**，而同步式训练迫使所有 GPU 等待最慢者。
- **根因分析**：注意力计算具有**输入依赖性**（input-dependent），且训练文档长度高度偏斜。长文档尾部 token 需 attend 更多前序 token，即使总序列长度相同，含长文档的序列计算量也显著更高。而现有 4D 框架把输入按固定长度打包、等量切分，**把所有 token 视为同质**，必然造成不均衡。
- **两级失衡**：
  - **PP 层**：微批次（micro-batch）间因固定长度打包而工作量不等；PP 的"生产者—消费者"数据依赖会把不均衡沿关键路径**放大**。
  - **CP 层**：整条序列被等长切块分给 CP worker，当序列内含多个文档时，分片间注意力负载差异巨大（TP 层因 AllGather 后各 worker 持有相同 chunk，故无失衡）。
- **已有方案不足**：仅靠 shuffle/repack 需跨多个 global batch 的大打包窗口，会破坏数据采样随机性、损害收敛；且无法解决 CP 层的**文档内**分片不均衡。

## 2. 方法论

### 2.1 总体思想
放弃"每个微批次固定 token 数"的静态假设，改为让每个 GPU 获得**总计算 + 通信工作量相等**的输入；并只对极少数超长文档调整执行顺序，以最小化对数据随机性的影响。

### 2.2 PP 层：负载感知变长打包 + 离群文档延迟

- **关键观察**（图 7）：注意力延迟随文档长度**二次**增长，而 GEMM、集合通信、element-wise 等操作随长度**线性**增长。因此可用多个短文档"补足"一个长文档的**总**延迟。
- **从定长到变长**：把优化目标从仅平衡注意力工作量 `min max Σ x_ij·d_i²`（定长打包，长度约束 Σx_ij·d_i ≤ S）改为平衡总工作量：
  `min max Σ ( W_a(x_ij·d_i) + W_l(x_ij·d_i) )`，约束 Σx_ij·d_i ≤ S_max；
  其中 `W_a(·)`、`W_l(·)` 分别为注意力与其他操作的延迟预测函数（离线 profiling 得到），`S_max` 为显存允许的最大序列长度。
- **离群文档延迟（Outlier Delay）**：设置**多级 FIFO 等待队列**，第 i 级队列阈值 `L_i`（L₁<L₂<…<L_{n+1}），长度落入 `[L_i, L_{i+1})` 的文档入对应队列；当某队列积累到 N（每轮微批次数）时弹出，使每个微批次分到等量离群文档。依据是长文档虽对失衡影响最大，却只占训练 token 的很小比例（<½上下文窗口的文档贡献 >75% token）。
- **超参 `L_i` 调优**：在小子集上采样，同时评估"均衡度"与"per-token 延迟"，取均衡高且延迟低的值。
- **启发式打包算法（Algorithm 1）**：输入 dataloader、队列 Q、微批次数 N、长度上界 S_max；流程为：分离离群文档入队 → 队列满则弹出 → 按长度降序排序 → 贪心放入"当前工作量最小"且不超 S_max 的微批次，否则放入"当前长度最小"的微批次，仍不行则留待下一轮。开销仅 ~20 ms/批。

### 2.3 CP 层：细粒度按文档分片 + 自适应选择

- **Per-Document Sharding**：现有 per-sequence 策略把整条序列切成 `2×CP_size` 块，第 i 个 worker 取第 i 与第 `2×CP_size−1−i` 块；多文档打包时失衡严重。新策略对**每个文档**切成 `2×CP_size` 块，每个 CP worker 取对称的一对，从而同时保证 token 数相等（GEMM 与通信均衡）与注意力工作量相等。
- **免 padding 设计**：设文档长 `d_i = e_i + r_i`，`e_i = ⌊d_i/(2×CP_size)⌋`；`e_i` 部分标准分片，余数 `r_i` 个 token 轮转分配。由于 `Σd_i` 与 `Σe_i` 均可被 `2×CP_size` 整除，故 `Σr_i` 亦然，各 worker token 数严格相等，无需 padding。
- **均衡 vs. 内核效率的权衡**（图 10）：① Tile 级浪费——FlashAttention tile=128，`Q_len<128` 仍按 128 计算，延迟不降；② TMA 效率——`Q_len≥256` 时多 thread block 共享同一 KV，可经 L2 与 TMA load multicast 降低搬运开销，TFLOPs 显著提升。故细粒度分片并非总是更快。
- **自适应分片选择**（图 11）：前向 AllGather 获得完整 KV 后，分别计算 per-sequence 与 per-document 两种方案下的 Q/KV 张量形状 → 按 FLOPs（含 pad 到 tile 倍数）÷ 离线 profiling 得到的达成 TFLOPs → 估算两种内核延迟 → 选择更低者。

### 2.4 系统实现
基于 FSDP（DP）、interleaved 1F1B（PP）并扩展为**变长 pipeline**、AllGather 式 CP（Llama3 风格）、1D TP + 序列并行，并做计算—通信重叠。

## 3. 实验设计

- **模型/场景**：内部 LLaMA-like 模型 550M、7B、30B、70B；上下文 64K 与 128K（敏感性分析另含 32K–160K）。7B 与 LLaMA2-7B 架构一致，其余按比例调整层数与维度。
- **并行配置**（表 1）：550M-64K 用 32 GPU (TP2,CP2,PP4,DP2)；550M-128K 用 32 GPU (2,4,4,1)；7B-64K 32 GPU (4,2,4,1)；7B-128K 64 GPU (8,2,4,1)；30B-64K 64 GPU (8,2,4,1)；30B-128K 128 GPU (8,4,4,1)；70B-64K/128K 均 256 GPU (16,4,4,1)。全局 batch = PP_size × DP_size，bfloat16。
- **Benchmark / 指标**：端到端训练加速比（相对 baseline）；微批次前向延迟衡量的失衡度；打包开销（ms/批）；训练 loss 曲线（收敛性）。
- **对比方法**：
  - **Plain-4D**：内部生产级 4D 代码库，不做打包优化，CP 用 per-sequence 分片；
  - **Fixed-4D**：定长打包 + 贪心（限定单个 global batch 以保随机性），CP 在 per-seq / per-doc 中取更优者；
  - 表 2 中另对比 **Fixed-Len Greedy**（1/2/4/8 个 global batch）、**Fixed-Len Solver**（ILP + Gurobi 求解）、**Original Packing**；
  - 图 15 中另对比 **Per-Seq**、**Per-Doc**、**Optimal**（逐微批次取两者最优）。
- **实验组构成**：8 组端到端配置；7B-128K 的逐项 speedup breakdown；5 个上下文窗口点的敏感性；打包方法与队列数的均衡度/开销对比（约 8 种配置）；CP 分片 4 策略 × 2 上下文；550M
