---
title: "Efficient {LLM} Serving on Commodity {GPU} Clusters with {Data-Reduced} {Cross-Instance} Orchestration"
title_zh: 在商用GPU集群上通过数据缩减的跨实例编排实现高效LLM服务
authors: "Jiangsu Du, Hongbin Zhang, Taosheng Wei, Zhenyi Zheng, Jiazhi Jiang, Kaiyi Wu, Zhiguang Chen, Yutong Lu"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-du.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 跨实例数据缩减编排降低集群GPU实例间的通信开销
tldr: 现有LLM服务要么将预填充与解码混部导致严重干扰，要么完全分离却依赖这类集群并不具备的高性能互连。EcoServe针对商用GPU集群提出部分分离策略，在单实例内分离预填充与解码，并在实例间进行数据缩减的协作编排以降低跨实例传输量。系统据此在有限互连带宽下缓解预填充-解码干扰。该方案为主流低成本AI基础设施提供了可落地的LLM服务部署路径。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 商用GPU集群缺乏高性能互连，现有LLM服务策略难以同时规避预填充与解码干扰和跨实例通信开销。
method: EcoServe提出部分分离策略，在单实例内分离预填充与解码，并在实例之间进行数据缩减的协作编排以减少传输。
result: 实验显示该方案在商用GPU集群上有效缓解预填充-解码干扰，并在有限互连条件下维持较高的服务效率。
conclusion: 该工作说明无需高速互连也能通过跨实例数据缩减实现高效LLM服务，降低了AI基础设施门槛。
---

## Abstract
Existing LLM serving strategies can be categorized by whether prefill and decode phases are disaggregated: non-disaggregated (NoDG) or fully disaggregated (FuDG). However, they neither fit commodity GPU clusters, which remain widely deployed as mainstream AI infrastructure. NoDG suffers from severe prefill–decode interference, while FuDG depends heavily on high-performance interconnects that such clusters lack. We present EcoServe, an LLM serving system tailored to commodity GPU clusters. It enables a data-reduced collaboration among inference instances to mitigate prefill-decode interference, termed the partially disaggregated (PaDG) strategy. Particularly, within a single instance, PaDG disaggregates the prefill and decode phases along the time dimension to mitigate interference and enhance throughput. Next, it coordinates multiple instances and cyclically activates them to ensure the continuous availability of prefill processing, thereby rescuing latency. Thus, EcoServe’s basic serving unit is the macro instance, within which multiple instances collaborate. It further incorporates an adaptive scheduling algorithm to route requests in a macro instance and a mitosis scaling approach for fine-grained capacity adjustments in online scenario. On a 32-GPU NVIDIA L20 cluster over Ethernet, EcoServe improves goodput by 1.96×, 1.99×, 2.51×, and 2.40× when serving 30B- and 70B-scale LLMs, compared to four representative NoDG and FuDG systems, vLLM, Sarathi, DistServe, and MoonCake. EcoServe remains competitive even on an NVIDIA H100 cluster with NVLink and InfiniBand. Our code is released at https://github.com/MLSysU/EcoServe .

---

## 论文详细总结（自动生成）

# EcoServe 论文中文结构化总结

## 一、核心问题与整体含义（研究动机与背景）

- **背景**：LLM 推理需集群级部署。H100、GB200 等高端 GPU 与 NVLink、InfiniBand 高速互连虽然受到研究界青睐，但成本高昂、生产环境更新周期慢，**缺少高性能互连的商用（commodity）GPU 集群仍是主流 AI 基础设施**（如 PCIe 互连 + 10Gbps 以太网）。论文的真实动机来自某头部互联网公司超级 App 的线上服务场景（L20 集群，致谢中提到腾讯微信团队）。
- **两类现有范式的两难**：
  - **NoDG（非分离）**：预填充（prefill）与解码（decode）同实例混部（vLLM、Sarathi、Orca 等）。prefill 是计算密集型、decode 是访存密集型，二者互相干扰：联合调度会拉高 TPOT，插入 decode 又会拉高 TTFT；decode 难以攒够大 batch 打满 GPU，吞吐受限。chunked prefill 可缓解但带来重复 KV 访问，且与流水并行（PP）兼容性差。
  - **FuDG（完全分离）**：prefill/decode 分置于不同实例（DistServe、MoonCake、Splitwise）。虽消除干扰，但需在实例间搬运海量 KV cache。论文实测：Llama-30B 在 8×A800 节点上 KV cache 生成速率达 **38.96 GB/s**，理论上就需 ≥400 Gbps 网络；在 L20 上为 9.796 GB/s。此外 FuDG 还存在**计算/显存双重负载不均衡、并行兼容性受限、工程复杂度高**三大问题。
- **核心矛盾**：现有策略"要么忍受干扰，要么依赖昂贵互连"，**均不适配商用 GPU 集群**。
- **论文含义**：提出 **EcoServe**，以"数据缩减的跨实例协作"替代"数据传输"，在不依赖高速互连的前提下同时缓解 prefill–decode 干扰并保住 TTFT/TPOT SLO，为低成本 AI 基础设施提供可落地的服务路径。

---

## 二、方法论

### 2.1 核心思想：部分分离（PaDG, Partially Disaggregated）策略

PaDG 在**单实例内按时间维度分离**两个阶段，在**实例间按时间循环协作**，从而在"零 KV cache 跨实例传输"的前提下获得接近 FuDG 的干扰抑制效果。

### 2.2 关键技术一：时序分离（Temporal Disaggregation）

- 与传统 FuDG"按实例分角色"不同，PaDG 把 prefill/decode **映射到同一实例的不同时间片**，每个阶段持续时间更长。
- 收益：① 降低 prefill–decode 切换频率 → 干扰减弱、吞吐提高；② KV cache 始终本地 → 无跨实例搬运。
- 代价与化解：实例处于 decode 窗口时新请求需等待，可能违反 TTFT。论文利用 **saved TPOT（时间裕度）** 概念——现代服务是"打字机式"输出，只要在时间窗内产出足够 token 即满足 TPOT，因此 decode 快于约束时可累积松弛时间，用于"打断 decode 插入 prefill"而不违约。

### 2.3 关键技术二：滚动激活（Rolling Activation）

- 多实例**错峰、循环**进入 prefill 阶段，保证任一时刻**至少一个实例处于 prefill 窗口**，新请求总能被立即接纳，从而保住 TTFT。
- 各实例需持续向 macro-instance 调度器上报 decode 进度、显存占用等状态，以支撑即时协同。

### 2.4 系统抽象与分层架构

- **Macro Instance（宏实例）** 是 EcoServe 的基本服务单元：一组协同的实例，内部完成滚动激活。
- **三级调度**：
  - *Instance Scheduler*：单实例内执行（prefill–decode 协调、设备编排、执行上层指令），策略上"prefill 优先"，但持续处理活跃 decode 并周期上报进度。
  - *Macro-Instance Scheduler*：聚合多实例状态，按 profile 结果与 SLO 派发请求（主–从/master–worker 模式）。
  - *Overall Scheduler*：按能力把请求分配给各 macro instance（论文重点在 macro instance 内部）。
- 实现上：同节点部署多个公共消息队列，实例周期性上传状态，调度器聚合后做全局决策。

### 2.5 关键技术三：自适应调度算法（Algorithm 1 约束检查）

宏观调度器**循环遍历**实例，对每个新请求先尝试复用上一请求的实例，若通过约束检查则转发，否则顺延下一个实例。约束检查三关：

1. **TTFT 约束**：统计本实例当前 prefill 窗口内所有待处理 prefill（含新请求）的总时长 `t_total`（单请求 prefill 时长可由不同长度序列的离线 profile 预测），若 `t_total > S_TTFT` 则拒绝。
2. **TPOT 约束**：对已存在的 decode 请求逐个计算 saved TPOT = `L × S_TPOT − (current_time − first_token_time)`，取均值；若 `mean_saved_TPOT < t_total` 则拒绝。
   - 关键设计：**不假设整个 decode 过程均匀借用裕度**，而是在**短解码区间内**局部累积、局部消耗，避免依赖长程预测，鲁棒性更好。
3. **显存约束**：新请求 KV cache 需求 > 剩余显存则拒绝。

### 2.6 关键技术四：有丝分裂式扩缩容（Mitosis Scaling）

- 动机：以 macro instance 为粒度扩缩容过粗；单 macro instance 管理全部实例会使调度器成为瓶颈。
- **两级动作**：先在 macro instance 内增删实例，再拆分/合并整个 macro instance。
- 超参 `Nl` / `Nu`：macro instance 内实例数下界/上界。`Nl` 太小导致频繁相位切换，`Nu` 太大带来管理开销与调度瓶颈。
- **扩张**：逐实例加入原 macro instance → 超过 `Nu` 时拆出一个含 `Nl` 实例的新 macro instance → 后续实例继续加在原/新 macro instance。**收缩**：从小 macro instance 移除至 `Nl` → 再从满 macro instance 移除 → 两者总数达 `Nu` 时合并为一。被移除实例先完成已分配请求再释放。常态是"多个满 macro instance + 1~2 个半满 macro instance"。
- **触发阈值**：
  - 扩张用 **TTFT SLO 达成率**（过载时 TTFT 最先、最敏感地下降，因为无实例满足约束时请求被挂起）：平均 TTFT 超过 SLO 即扩容（式 1）。
  - 收缩用 **saved TPOT**：合理区间为 `[T_TTFT, T_TTFT·(n+1)/n]`，其中 `(n+1)/n` 反映"若从 n 扩到 n+1 实例后单实例 decode 负载的下降比例"；超过上界即判定过配并缩容（式 2）。
- **可序列化代理对象（InstanceHandler）**：封装实例入口与公共消息队列，用 pickle 序列化后在 macro instance 调度器间迁移句柄，实现**不中断执行、不重新初始化模型**的逻辑迁移。实测重初始化 CodeLlama2-34B 需约 3 分钟（本地存储），远程更久。

### 2.7 指标定义上的一个细节

- 论文指出 TTFT/TPOT 作为 runtime 侧指标不足以刻画体验：请求进入 decode 前还存在**相位切换等待**（NoDG/PaDG 是其他请求的 prefill 排队，FuDG 是 KV cache 传输）。EcoServe 报告的 TTFT **包含**该等待时间，因而构成**更严格的 SLO 定义**，TPOT 计时从切换完成后开始。

---

## 三、实验设计

### 3.1 测试集群（3 套，覆盖商用与高性能互连）

| 集群 | 规模 | GPU | 互连 |
|---|---|---|---|
| L20（主） | 8 节点 × 8 = 64 GPU | L20-48GB | PCIe + 10 Gbps 以太网 |
| A800 | 2 节点 × 8 = 16 GPU | A800-80GB | PCIe + 25 Gbps RoCE |
| H100 | 2 节点 × 8 = 16 GPU | H100-80GB | 节点内 NVLink + 8×400 Gbps InfiniBand |

### 3.2 模型与数据集

- **模型**：Llama-30B（MHA）、CodeLlama2-34B（GQA）、Qwen2-72B
