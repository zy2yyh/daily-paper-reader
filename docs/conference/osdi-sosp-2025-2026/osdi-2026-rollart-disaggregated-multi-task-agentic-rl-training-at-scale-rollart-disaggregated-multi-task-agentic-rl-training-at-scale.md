---
title: "{RollArt}: Disaggregated {Multi-Task} Agentic {RL} Training at Scale"
title_zh: RollArt：面向大规模解耦式多任务智能体强化学习训练
authors: "Wei Gao, Yuheng Zhao, Tianyuan Wu, Shaopan Xiong, Weixun Wang, Dakai An, Lunxi Cao, Dilxat Muhtar, Zichen Liu, Haizhou Zhao, Ju Huang, Siran Yang, Yongbin Li, Wenbo Su, Jiamang Wang, Lin Qu, Bo Zheng, Wei Wang"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-gao.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: 解耦式强化学习训练中的跨阶段同步与硬件路由
tldr: 针对多任务智能体强化学习训练中预填充、解码、环境执行与奖励评估负载异构且跨阶段同步开销大的问题，RollArt 将各流水线阶段映射到最合适的异构硬件，并在解耦式基础设施上拆分 rollout 流程。系统通过按任务特性路由计算与带宽资源，降低阶段间协调成本。实验表明该设计能提升大规模强化学习训练的资源利用率与整体效率，为异构集群上的分布式训练调度提供了新思路。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-rollart-disaggregated-multi-task-agentic-rl-training-at-scale/fig-001.webp\", \"caption\": \"\", \"page\": 13, \"index\": 1, \"width\": 2371, \"height\": 845}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-rollart-disaggregated-multi-task-agentic-rl-training-at-scale/fig-002.webp\", \"caption\": \"\", \"page\": 13, \"index\": 2, \"width\": 2371, \"height\": 845}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-rollart-disaggregated-multi-task-agentic-rl-training-at-scale/fig-003.webp\", \"caption\": \"\", \"page\": 13, \"index\": 3, \"width\": 2371, \"height\": 847}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-rollart-disaggregated-multi-task-agentic-rl-training-at-scale/fig-004.webp\", \"caption\": \"\", \"page\": 13, \"index\": 4, \"width\": 514, \"height\": 514}]"
motivation: 现有多任务智能体强化学习系统粗粒度解耦或同置部署，忽视硬件异构并带来显著跨阶段同步开销。
method: RollArt 将预填充、解码和环境等流水线阶段映射到最适硬件，并在解耦基础设施上拆分 rollout 调度。
result: 通过按负载特性路由计算与带宽资源，系统降低了阶段间协调成本并提升了大规模训练的资源利用率。
conclusion: 该工作表明面向异构硬件的细粒度阶段解耦可有效支撑大规模多任务智能体强化学习训练。
---

## Abstract
Agentic Reinforcement Learning (RL) trains LLMs through multi-turn interactions with environments, producing workloads that mix compute-bound prefill, bandwidth-bound decoding, CPU-heavy environment execution, and bursty reward evaluation. Existing systems either colocate all stages on a single GPU cluster or decouple them only at a coarse granularity, overlooking hardware heterogeneity and incurring substantial synchronization overhead across stages. We present ROLLART, a system for multi-task agentic RL on disaggregated infrastructure. ROLLART maps each pipeline stage to best-fit hardware, routing prefill-heavy tasks to compute-optimized GPUs, decode-heavy tasks to bandwidth-optimized GPUs, and environments to CPU clusters. It decouples rollout at the trajectory level, allowing generation, environment interaction, and reward scoring to proceed independently, so that slow or failed environments never block the others. ROLLART offloads stateless reward computation to serverless infrastructure and overlaps rollout with training via staleness-bounded asynchronous weight synchronization. Our results demonstrate that ROLLART effectively improves training throughput and achieves 1.31–2.05× training time reduction compared to various RL systems. We also evaluated ROLLART by training a hundreds-of-billions-parameter MoE model for Qoder product on an Alibaba cluster with above 3,000 GPUs, demonstrating its stability and scalability.

---

## 论文详细总结（自动生成）

# RollArt：解耦式多任务智能体强化学习训练系统 —— 论文深度总结

## 一、核心问题与研究背景

- **研究范式转变**：强化学习正把 LLM 从单轮推理推向长程、自主的决策（agentic RL）。智能体需与外部环境（工具调用、网页导航、计算机控制）进行多轮交互，形成一条长轨迹（trajectory），再经奖励打分后用于训练。
- **三阶段流水线**：rollout（多轮生成 + 环境交互）→ reward（规则脚本、代码沙箱或 LLM-as-a-Judge 打分）→ training（PPO/GRPO 更新权重，并同步回 rollout）。
- **核心矛盾——资源异构性贯穿阶段内外**：
  - *阶段内部*：LLM 生成由 **计算受限的 prefill** 与 **带宽受限的 decode** 交替组成，二者比例随任务域变化。实测在成本等价硬件上，H800 把 prefill 重的任务耗时压到 H20 的 0.53×，而 H20 把 decode 重的任务压到 H800 的 0.49–0.79×。
  - *环境侧*：有状态、CPU 受限、长尾极重（`env.reset` 在生产中可达数百秒，源于镜像拉取与宿主机 I/O 争用），且故障频发（约每 10 次迭代出现一次环境失败，失败时单步耗时从 366s 涨到 513s，`env.reset` 独占 rollout 时间 78%）。
  - *奖励侧*：无状态、突发性强、专用 GPU 利用率低至 **7.4%**（图 6）。
  - *训练侧*：需要高端 GPU 与高速互联。
- **现有系统的不足**：
  - 单体框架（veRL、slime、rLLM）把全部阶段同置在单一 GPU 集群，完全忽略异构性。
  - 部分解耦系统（AWorld、DeepSWE）只把环境卸载到 K8s，rollout 与 training 仍同置。
  - 异步系统（StreamRL、AsyncFlow、SeamlessFlow）只拆训练/rollout，把 rollout 当作整体，忽视 prefill/decode 分化；AReaL 引入无界陈旧性异步，但环境交互仍是批式、奖励仍与生成 GPU 耦合；Laminar 做轨迹级调度，但不利用 rollout 内的硬件亲和性、也不卸载无状态奖励。
- **整体含义**：论文主张不存在单一硬件能同时满足所有阶段，必须把流水线拆解并按"最适配硬件"路由，同时用轨迹级异步 + 有界陈旧性来掩盖解耦带来的同步开销。据此提炼出四条需求 **R1–R4**：
  - **R1** 硬件亲和的工作负载映射（含 rollout 内部的子阶段）；
  - **R2** 轨迹级异步 rollout（慢/失败环境不阻塞他人）；
  - **R3** 无状态奖励的 serverless 卸载；
  - **R4** 有界陈旧性的异步训练（训练与 rollout 并行）。

## 二、方法论

### 2.1 总体架构：三层平面

- **资源平面（Resource Plane）**：`resource manager` 通过共享元数据存储（如 Redis）维护 H800/H20/CPU/serverless 端点等异构池的全局实时视图，依据 Worker 的注解把角色绑定到资源；首选资源不可用时**机会式回退**到兼容默认资源，避免部署停顿。
- **数据平面（Data Plane）**：`pipeline runner` 构造训练、生成、环境、奖励四类 `Cluster`，各自拉起 `Worker` 并绑定后端运行时（Megatron、vLLM）。
- **控制平面（Control Plane）**：`rollout scheduler` 调度轨迹级 rollout；辅以 `LLMProxy`（生成请求中介）与 `SampleBuffer`（已打分轨迹缓冲），使 rollout、reward、训练三者并发推进。

### 2.2 编程模型：Worker / Cluster 抽象 + 三个装饰器

- **Worker**：跨资源与数据平面的基本执行单元，封装用户计算逻辑与方法级声明，可派生出训练/生成/奖励/环境角色。
- **Cluster**：Worker 组的代理与控制器，负责创建、方法绑定、集合调用与结果聚合（`execute_all` 通过 `ray.get` 汇总）。
- **三个 Python 装饰器**（Listing 1）：
  1. `@register(mode="execute_all")`：单控制器模型，广播调用并聚合返回值（典型如 `compute_gradients`）。
  2. `@hw_mapping(hw_affinity={"FrozenLake": "H800", "default": "H20"})`：声明任务域到硬件类的亲和映射；运行时把当前任务标识作为 `tag_name` 传入，据此过滤并路由请求。**刻意做成任务域级粗粒度**，而非逐请求负载均衡。
  3. `@register_serverless(attribute, serverless_url)`：把无状态奖励函数重定向到 serverless 平台（如 Function Compute），以纯函数方式调用。
- **Cluster 侧实现**（Listing 2）：按 `tag_name` 查表得到 `hw_type`，筛出匹配资源类型的 Worker 并路由；serverless 场景把目标属性替换为调用远端 URL 的可调用对象；所有亲和路径都带降级回退。

### 2.3 轨迹级 rollout 与奖励（对应 R2/R3）

- **LLMProxy**：解耦生成客户端与服务实例，按**单条轨迹**粒度分发请求。推理 Worker 运行命令驱动的事件循环：
  - *Step-wise 命令处理*：在引擎步之间轮询 `ADD`（入队）/`ABORT`（取消）命令，无命令时推进 prefill/decode 步，保证 GPU 利用率；增删轨迹不打断在途生成。
  - *后处理回调*：请求完成即刻触发预注册回调，把结果直接回送对应的 `EnvManager`，无需等待同批其他轨迹。
- **EnvManager**：每个环境一个轻量控制器，独立事件循环，维护 `(observation, action)` 列表；与批式交互（图 5b，快环境必须等最慢者）不同，各环境各按自己的时间线推进。
- **重叠**：轨迹一完成即异步派发到 serverless 奖励函数，奖励延迟被后续 rollout 掩盖；同时避免奖励与生成争抢 GPU 显存/KV cache/调度槽位。

### 2.4 异步训练编排与权重同步（对应 R4）

六步同步协议（图 9）：
1. **get_batch**：阻塞式从 `SampleBuffer` 取轨迹，直到凑满预设 batch。
2. **suspend**：向 LLMProxy 发挂起命令，停止接收新请求但保留在途轨迹。
3. **update**：从训练 Worker 取最新权重并更新全部推理 Worker。
4. **resume**：恢复 LLMProxy，继续挂起的生成请求。
5. **recomp & rollout**：对在途轨迹做 **KV cache 重计算**，在新权重下重建 KV，使生成可续跑而非从头开始。
6. **train_step**：与恢复后的 rollout 并行执行训练步。

**有界陈旧性**：设当前模型版本为 `n`，缓冲轨迹的起始版本不得早于 `n − α`，超窗轨迹被中止；`SampleBuffer` 规模上界为 `O(α·E)`（E 为并发环境数），`get_batch` 前主动驱逐陈旧轨迹，防止缓冲无界增长。与 AReaL 只在轨迹起点约束陈旧性不同，RollArt 在**每次迭代内**做轨迹级陈旧性控制，避免长尾轨迹跨越多个模型版本。默认 `α = 1`。

### 2.5 跨平面优化

- **数据搬运分层**：轨迹/监督信号以 Ray object reference 流式传递并按 Worker 并行布局分片；簇内权重同步用 NCCL over NVLink/IB；**跨簇**用基于 Mooncake 的异步权重更新引擎——训练后按约 1GB 分桶写入远端 CPU 常驻 Mooncake 存储（低带宽跨簇链路只写一次），推理 Worker 按需经高带宽簇内链路异步拉取，避免同步推送造成的 GPU 停顿。
- **冗余环境 rollout**：启动多于所需数量的环境，凑够目标轨迹数后终止在途轨迹；利用 GRPO 的组结构掩盖环境失败与长尾。
- **PD 解耦**：把 prefill 与 decode Worker 分别部署在计算优化与带宽优化 GPU 上，将亲和路由从任务级细化到阶段级。

## 三、实验设计

### 3.1 场景与数据集

- **任务环境**（表 1）：SWE-bench（软件工程，30–50 轮，prefill 重）、WebShop（电商，5–30 轮）、FrozenLake（游戏，20–100 轮，文本+视觉）、GEM-math（数学+工具，<5 轮，decode 重）、GEM-game（1 轮）。
- **模型**：Qwen3 8B/14B/32B，最大上下文 32k；SWE-bench 因难度大只在 Qwen3-32B 上训练；数学任务用 Qwen2.5-7B 作奖励 LLM。
- **训练配置**：GRPO，batch size 512，group size 8，任务均匀采样；rollout TP 度分别为 1/2/4；vLLM 0.8.4（开 prefix caching 与 CUDA graph），Megatron 0.12.2。
- **指标**：平均 step time（5 次迭代）、吞吐（全局 batch 的 prompt+response token 数 / step time）、任务平均验证分；端到端用 **time-to-score（目标 0.85）**。

### 3.2 对比基线

- **Sync**：标准同步 RL 流水线。
- **Sync+**：在 Sync 上叠加异步奖励、异步环境交互、serverless 卸载（近期系统广泛采用的强化版同步基线）。
- **One-off**：消费上一步轨迹的异步方案（要求整轮轨迹在陈旧权重下完成）。
- **AReaL**：在 RollArt 代码库内按其陈旧性界（=1，仅在轨迹起点约束）重实现。
- **Laminar**：闭源无法直接跑，改用"特性分解"论证——分别测量 RollArt 相对 AReaL 的硬件亲和增益、环境不稳定性下的 1.23–2.27× 增益、serverless 奖励的约 2× 增益，以此下界估计差距。

## 四、资源与算力

- **评测集群**：96 卡 H800 集群 + 32 卡 H20 集群；簇内 400 Gbps InfiniBand，跨簇 200 Gbps 以太网；SWE-bench 与其余环境各用独立 CPU 集群；奖励用内部 serverless 平台。**除非特别说明，实验统一使用 128
