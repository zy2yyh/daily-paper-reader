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
score: 5.0
evidence: 基于增量的通信组建立以快速恢复分布式训练
tldr: 大规模ML训练常因硬件故障与管理事件中断，检查点重启和运行时重配置带来长时间停机与性能损失。TrainMover构建弹性容错训练运行时，采用两阶段增量式通信组建立、无通信沙箱预热和通用备用角色设计实现快速恢复。评测显示其在多种中断下停机时间约20秒且零额外内存开销。该工作为超大规模训练作业的高可用运行提供了实用机制。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 大规模ML训练频繁因硬件与软件异常中断，检查点重启或运行时重配置会导致长时间停机与性能下降。
method: 利用弹性与备用机器，采用两阶段增量式通信组建立、无通信沙箱预热和通用备用设计实现快速恢复。
result: 在多种中断场景下将停机时间稳定控制在约20秒，且不引入额外内存开销。
conclusion: 表明通信组快速重建与备用机制结合可显著提升大规模训练作业的容错效率。
---

## Abstract
Large-scale ML training jobs are frequently interrupted by hardware and software anomalies, failures, and management events. Existing solutions like checkpoint-restart or runtime reconfiguration suffer from long downtimes and degraded performance. We present TrainMover, a resilient LLM training runtime that leverages elastic and standby machines to handle interruptions with minimal downtime and zero memory overhead. To achieve these goals, TrainMover introduces three key techniques: two-phase, delta-based communication group setup; communication-free sandboxed warmup; and general standby design that enables failure recovery from any role. Our evaluation shows that TrainMover consistently achieves around 20 seconds of downtime when handling various interruptions at the 1024-GPU scale. TrainMover is projected to reduce wasted GPU hours by 55% compared to the best alternative, saving 1.4 million GPU-hours per week at the 64K-GPU scale.

---

## 论文详细总结（自动生成）

# TrainMover 论文总结

## 一、核心问题与整体含义（研究动机与背景）

- **背景**：大语言模型（LLM）训练作业通常部署在数千至十万级 GPU 上、持续数周至数月，依赖 Megatron-LM、DeepSpeed、NeMo 等紧耦合同步框架。规模越大，中断越频繁，单点故障即可拖慢或终止整个作业。
- **中断来源**（论文归纳为四类）：
  - 硬件异常（GPU 故障、过热/功耗降频；H100 的 MTTF 仅为 A100 的一半）；
  - 软件争用（同机程序干扰、OS 噪声、Python GC；Falcon 报告平均拖慢 34.59%）；
  - 网络故障（光模块/交换机故障、拥塞；40–50% 作业经历网络导致的 fail-slow）；
  - 管理事件（驱动更新、资源再平衡、功耗/热规划；Llama-3 每天至少一次此类中断）。
- **中断分类**：预期/计划中断（有提前通知，作业仍存活但性能下降）与非预期中断（GPU/CPU 故障导致框架立即崩溃）。
- **核心痛点**：
  - 现有最佳实践是“停止–重调度–重新初始化”（checkpoint-restart）。8192-GPU 生产作业每次中断平均 6.47 分钟停机，其中框架初始化占 4.45 分钟（checkpoint 加载 35.1%、NCCL 实例化 24.5%、冷启动预热 40.4%）。
  - 运行时重配置系统（Oobleck、Parcae、ReCycle）虽可避免全局重启，但新增 joiner 仍需完整冷启动：Oobleck `+1` 超过 100 秒、Parcae 超过 200 秒，关键路径并未缩短。
  - 由于训练布局（并行配置、模型切分）为峰值性能高度定制，简单改变布局会导致性能下降、OOM 或 GPU 空闲，因此业界普遍保留约 6% 备用 GPU 池（Alibaba HPN、ByteDance、Google、Meta 等）。
- **量化影响**：ETTR（有效训练时间比）在 8192 GPU 时降至 0.6；64K/128K GPU 规模下吞吐损失 16.47%/31.79%，对应每天 95 万/363 万美元损失。
- **目标**：在保持训练布局不变的前提下，把 joiner 上线所需的重型初始化移出关键路径，实现秒级停机与零额外 GPU 内存开销，从而维持高 ETTR。

## 二、方法论

### 1. 核心思想
- 只把受影响机器（leaver）的工作迁移到健康备用机（joiner），其余机器（stayer）保持不变。
- 将迁移拆为两个阶段：
  - **准备阶段（Preparation Phase）**：joiner 在后台完成初始化与通信组准备，与前台训练重叠，不影响训练；
  - **切换阶段（Switching Phase）**：当前迭代结束后，joiner 一对一接收 leaver 的最新状态并接管其角色。
- 三大支撑技术：无通信沙箱预热（§4）、两阶段增量式通信组建立（§5）、通用备用设计（§6）。

### 2. 关键技术细节

**(1) 无通信沙箱预热（Sandboxed Initialization）**
- **问题**：现代框架的初始化是隐式、跨层（PyTorch/NCCL/CUDA）、与通信交织的（如 PP 下逐 stage 依赖 P2P 通信），难以单独提前触发。
- **方案**：让 joiner 在隔离沙箱中独立运行一次“影子迭代”（shadow iteration），自然触发全部隐式初始化。
  - **预记录（Pre-record）**：在作业最开始的第一（或少数几）次迭代，用 hook 拦截 PyTorch 与 CCL 之间的集合通信调用，把所有集合通信的输出张量记录到持久存储；之后移除 hook，后续训练零开销。
  - **重放（Replay）**：joiner 从 checkpoint 加载初始状态，进入沙箱运行影子迭代；拦截所有向外发起的通信调用，用已记录张量作为响应；`barrier`、`send` 等不影响调用者状态的操作直接旁路。
- **正确性保证**：切换阶段 leaver 的最新模型/优化器状态会覆盖沙箱产生的本地状态；框架的确定性（固定 kernel 调度、静态内存布局、确定性集合通信顺序）保证初始化被正确触发。MoE 路由等动态路径未被覆盖时，其影响仅为毫秒级，在切换阶段补齐。
- **低开销设计**：只记录可能缺失的“边界”通信；机器内通信（如 TP 的 NVLink 流量）可在沙箱内原生运行；重复角色可去冗余记录；GPT-5.12T MoE 下存储开销可降至 300 GB 以内。

**(2) 两阶段增量式 CCL 建立（Two-phase Delta-based CCL Setup）**
- **问题**：CCL 建立慢（网络 bootstrap、拓扑发现、连接建立；64×A100 上约 50 秒），且组成员静态、修改需全局销毁重建；后台预分配新组又会带来 GPU 内存开销。
- **Phase 1（与训练重叠）**：
  - Stayer 调用 `CCL_prepare_stayers(new_topology)` 增量修改现有组；joiner 并发调用 `CCL_prepare_joiners()`；原组通信保持可用。
  - 复用已有 TCP bootstrap 连接；新增连接与拓扑状态存于 CPU 内存，**无 GPU 内存开销**。
  - 交换拓扑信息后本地计算 **delta 拓扑**，生成 delta 重配置计划：机器内通道（NVLink）通常完全继承；仅少量跨机通道需更新，推迟到 Phase 2。
  - 完成后进入 `ready-to-switchout` 状态，训练不中断。
- **Phase 2（切换）**：调用 `CCL_switchover()`，在 RDMA 队列对（QP）级别把跨机连接从 stayer–leaver 切换为 stayer–joiner，回收旧拓扑资源。这是**唯一贡献网络停机**的步骤。
- 设计兼容分层拓扑、SHARP 等多种通信技术，且全程零额外内存开销。

**(3) 通用备用（General Standby）**
- **依据**：LLM 训练 rank 具有强对称性——TP/DP/EP 等并行方式下各 rank 的参数形状、优化器布局、算子集合、CUDA kernel 与集合通信模式相同。
- **PP 例外**：至多三种角色类型——首 stage（含 embedding）、中间 stage（重复 transformer 块）、末 stage（含 output 层）；首末层形状相同（权重共享）。
- **流程**：standby 顺序独立运行最多 3 次预热迭代覆盖三种角色，之后保留中间 stage 角色；若故障发生在首/末 stage，只需额外分配少量层（embedding/output）的参数与优化器状态 delta，其余 kernel 已编译。
- **资源**：1–2 台 standby 即可覆盖所有角色类型；同时双机故障概率极低（∝1/MTTF）。也支持无 standby 场景，走重叠恢复路径。

**(4) 状态同步与工作流**
- 预期中断：controller 发信号 → 选弹性机为 joiner → 后台准备（§4、§5）→ 切换阶段 leaver 一对一传输最新状态（覆盖沙箱状态）→ 接管。
- 非预期中断：热 standby 已在后台完成准备 → 检测故障 → 提升为 joiner → 切换阶段从邻居冗余副本或远程 checkpoint 取状态，其他机器回滚至同一 checkpoint。
- 状态传输为一对一
