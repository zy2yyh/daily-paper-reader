---
title: "{TrainMover}: An {Interruption-Resilient} Runtime for {ML} Training"
title_zh: TrainMover：面向机器学习训练的抗中断运行时
authors: "ChonLam Lao, Jiaqi Gao, Jiamin Cao, Zhipeng Zhang, Pengcheng Zhang, Jiangfei Duan, Zhilong Zheng, Yu Guan, Yichi Xu, Yong Li, Zhengping Qian, Aditya Akella, Minlan Yu, Ennan Zhai, Dennis Cai, Jingren Zhou"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-lao.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向分布式训练通信组的增量式两阶段通信组建立
tldr: 大规模ML训练常因硬件与软件异常、故障和管理事件中断，现有checkpoint-restart与运行时重配置方案停机时间长且性能下降。TrainMover提出弹性与备用机结合的容错运行时，采用两阶段增量式通信组建立、免通信的沙箱预热以及可从任意角色恢复的通用备用设计。评测显示其处理各类中断时停机时间约为20秒，且不引入额外内存开销。该工作降低了训练中断代价，其中通信组快速重建机制对分布式训练通信管理具有借鉴意义。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 大规模ML训练频繁被硬件软件异常和管理事件中断，现有重启与重配置方案停机久、性能差。
method: TrainMover结合弹性机与备用机，引入增量式两阶段通信组建立、免通信沙箱预热和通用备用角色设计。
result: 在多种中断场景下将停机时间稳定控制在约20秒，且不增加内存开销。
conclusion: 该运行时显著提升了训练作业的抗中断能力，并为通信组快速重建提供了可行做法。
---

## Abstract
Large-scale ML training jobs are frequently interrupted by hardware and software anomalies, failures, and management events. Existing solutions like checkpoint-restart or runtime reconfiguration suffer from long downtimes and degraded performance. We present TrainMover, a resilient LLM training runtime that leverages elastic and standby machines to handle interruptions with minimal downtime and zero memory overhead. To achieve these goals, TrainMover introduces three key techniques: two-phase, delta-based communication group setup; communication-free sandboxed warmup; and general standby design that enables failure recovery from any role. Our evaluation shows that TrainMover consistently achieves around 20 seconds of downtime when handling various interruptions at the 1024-GPU scale. TrainMover is projected to reduce wasted GPU hours by 55% compared to the best alternative, saving 1.4 million GPU-hours per week at the 64K-GPU scale.

---

## 论文详细总结（自动生成）

# TrainMover 论文结构化总结

## 1. 核心问题与研究背景

- **背景**：LLM 训练任务规模达数千至十万+ GPU，运行数周至数月（GPT-3 175B 在 1024 GPU 上约 34 天；Llama 3 405B 用最多 16000 H100 训练 54 天），且采用全同步紧耦合训练，任一 GPU 变慢或故障都会拖累整个集群。
- **中断频发**：
  - 硬件异常（GPU 故障、过热/功耗降频）；Llama 3 训练的 MTTF 仅 2.7 小时，Alibaba 报告 60% 大任务受慢节点影响、平均 JCT 增加 35%。
  - 软件争用（CPU 共置、OS 噪声、Python GC），可致 34.59% 平均减速。
  - 网络故障（光模块/交换机故障、拥塞），40–50% 大任务经历网络导致的 fail-slow。
  - 管理事件（驱动升级、安全补丁、资源共享与功耗均衡），Llama 3 每天至少一次此类中断。
- **核心矛盾**：为保持峰值性能，训练布局（并行配置与模型切分）被精心调优，不能随意改变；因此业界只能预留备用/弹性 GPU 池（Alibaba 6%、ByteDance、Google、Meta 均有），但**把新机器（joiner）重新拉上线必须重新初始化，这一关键路径始终未被消除**。
- **现有方案的两类局限**：
  - **Stop–reschedule–reinitialize（重启）**：8192-GPU 生产任务每次中断平均停机 6.47 分钟，其中框架初始化 4.45 分钟（checkpoint 加载 35.1%、NCCL 实例化 24.5%、冷启动 40.4%）。
  - **运行时重配置（Oobleck / Parcae / ReCycle）**：可 -1/+1 弹性伸缩，但 joiner 仍需完整冷启动——Oobleck -1 需 57 s、+1 超 100 s；Parcae -1 需 21 s、+1 超 200 s，甚至超过同规模 Megatron-LM 的整体初始化开销。
- **量化影响**：以 4.45 分钟重启时间结合 Meta 的 MTTF 外推，64K GPU 规模损失 16.47% 吞吐（约 $0.95M/天），128K 规模损失 31.79%（约 $3.63M/天）。
- **目标**：在**不改变训练布局、不增加 GPU 内存开销**的前提下，把中断处理停机压到秒级，维持高 ETTR。

## 2. 方法论

### 2.1 核心思想

- 将一次机器替换拆成**准备阶段（Preparation）**与**切换阶段（Switching）**：所有可预先完成的工作（框架初始化、CCL 建立、状态预热）都在后台与前台训练重叠执行；切换阶段只做最小化的连接切换与状态同步，从而把停机压到秒级。
- 对**预期中断**：收到迁移信号后选弹性机作 joiner 后台准备，前台训练不中断；准备完成后进入短暂切换阶段，joiner 从配对的 leaver 一对一接收最新状态并接管角色。
- 对**非预期中断**：预先在后台预热的**通用备用机**直接被提升为 joiner，进入切换阶段，从 checkpoint 或邻居/DP 冗余/远程 CPU 内存恢复状态。

### 2.2 关键技术一：免通信沙箱预热（§4）

- **问题**：现代框架（Megatron-LM、DeepSpeed、NeMo）的初始化是隐式、跨层（PyTorch/NCCL/CUDA）、且与通信交织的（如 PP 各 stage 串行依赖），无法单独提前触发。GPT-10B 实验中单次中断触发约 150 秒预热，首个迭代因 JIT 编译慢约 6 倍。
- **方案：合法张量 record–replay**
  - **Pre-record**：在 PyTorch 与 CCL 之间加 hook，在训练最初一个（或几个）迭代拦截所有集合通信的输出张量并落盘；之后移除 hook，后续训练零额外开销。
  - **Replay**：joiner 进入沙箱，先从 checkpoint 载入初始状态，再跑一次"影子迭代"；hook 拦截所有试图越出沙箱的通信调用并用记录张量作答；`barrier`、`send` 等不影响调用者状态的操作直接旁路。影子迭代结束后 joiner 即可在主循环中无缝接管。
- **正确性依据**：现代 LLM 训练为可复现性与峰值性能维持固定 kernel 调度、静态内存布局、确定性集合通信顺序，因此几乎所有初始化（data-loader、CUDA context、内存分配）都会被触发；MoE 路由等输入相关路径因框架预分配固定 expert buffer 而影响极小，遗漏部分在切换阶段以 ms 级补齐；leaver 的最新状态最终会覆盖 joiner 的全部本地状态。
- **低开销优化**：机器级迁移时 intra-machine 通信（TP 流量）无需记录；重复角色去重；批量迁移时 joiner 内部通信保持真实，只加载跨边界通信。GPT-5.12T MoE 下存储开销可降至 300 GB 以内。

### 2.3 关键技术二：两阶段增量式 CCL 建立（§5）

- **问题**：CCL 建立慢（8 机 64×40GB A100、GPT-10B、TP4/PP2/DP8 下约 50 秒：网络 bootstrap 2.48 s、拓扑发现 9.40 s、机内连接 21.49 s、机间连接 17.07 s），且成员静态、修改需全局销毁重建；后台预分配新组虽可隐藏时延但会占用额外 GPU 内存。
- **方案**：
  - **Phase 1（与训练重叠）**：stayer 调用 `CCL_prepare_stayers(new_topology)`，joiner 调用 `CCL_prepare_joiners()`；复用既有 TCP bootstrap 连接，交换拓扑信息，本地计算 **delta 拓扑** 与增量重配置计划，建立可本地完成的机内连接（NVLink 通常整体继承）。这些状态驻留 CPU 内存，**不占 GPU 内存**，原组通信保持可用。
  - **Phase 2（切换）**：调用 `CCL_switchover()`，在 **RDMA Queue Pair 级别**把 delta 的机间连接从 stayer–leaver 改为 stayer–joiner，回收旧拓扑资源。这是唯一的网络停机来源，且只影响 leaver 的邻居，无全局协调。

### 2.4 关键技术三：通用备用机（§6）

- **依据**：TP/DP/EP 等并行方式保持各 rank 参数形状与优化器状态布局一致、算子与集合通信模式相同（强对称性）；PP 是例外，但最多只有三类角色——first（含 embedding）、middle（重复 transformer block）、last（含 output layer），且 first/last 结构相似。
- **流程**：standby 在后台依次为 first / middle / last 各跑一次预热迭代（无 PP 时一次即可），保留 middle 角色；若故障发生在 first/last，只需补少量 delta 层（embedding/output 的参数与优化器状态）的分配，kernel 等已预编译。standby 停留在 CCL Phase 1，显存占用比峰值低数百 MB，足以容纳各角色预热产物（每角色仅数百 KB）。
- **效果**：单台 standby 可恢复任意角色，1–2 台即可覆盖 PP 各 stage 类型；也支持无 standby 的降级路径。作者用对称性把"每角色一台备用机"的需求降为"一台通用机"。

### 2.5
