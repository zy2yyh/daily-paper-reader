---
title: "{BlitzScale}: Fast and Live Large Model Autoscaling with O(1) Host Caching"
title_zh: BlitzScale：基于O(1)主机缓存的大模型快速热自动扩缩容
authors: "Dingyan Zhang, Haotian Wang, Yang Liu, Xingda Wei, Yizhou Shan, Rong Chen, Haibo Chen"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-zhang-dingyan.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 通过GPU间计算网络与网络优化组播加载参数
tldr: 大模型自动扩缩容受限于扩缩速度与参数缓存内存的权衡，数据面加载慢且实例在加载时停滞。BlitzScale利用GPU间计算网络传输参数，其速度接近主机缓存且利用率低，并用网络优化的组播支持多实例扩缩。由此数据面无需或仅需O(1)缓存即可快速加载，实现跨主机的热扩缩容。该工作为无服务器模型服务提供了可扩展的高效参数分发机制。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
motivation: 模型自动扩缩容面临扩缩速度与参数缓存内存的权衡，数据面加载慢导致实例停滞。
method: BlitzScale借助GPU间计算网络与网络优化组播加载参数，实现无需或O(1)缓存的数据面。
result: 数据面加载速度接近主机缓存，支持跨多主机的频繁热扩缩容。
conclusion: 复用计算网络与组播使大模型自动扩缩容兼顾速度与内存开销。
---

## Abstract
Model autoscaling is the key mechanism to achieve serverless model-as-a-service, but it faces a fundamental trade-off between scaling speed and storage/memory usage to cache parameters, and cannot meet frequent scaling requirements across multiple hosts. The key problem is that data plane performance is slow, and scaled instances remain stopped while parameters are loading. In this paper, we first show that the data plane can be made fast with no or O(1) caching by loading parameters through the compute network between GPUs because: (1) its speed is comparable to host cache and is underutilized, and (2) scaling multiple instances requires no or O(1) caching with network-optimized multicast. Second, autoscaling can be made live by breaking the scaling abstraction for inference from a coarse-grained instance-level to a fine-grained layer-level. This allows us to offload the layer computation from the overloaded serving instances to the scaled ones without waiting for the parameters to be fully loaded. Under real-world workloads, our system BLITZSCALE achieves up to 94 % lower tail latency reductions compared to state-of-the-art autoscaling system (ServerlessLLM), and it reduces the GPU time used for serving by 49 % when compared with serving systems that do not support autoscaling like DistServe and vLLM with the same service-level-agreement.

---

## 论文详细总结（自动生成）

# BlitzScale 论文深度总结

## 一、核心问题与整体含义（研究动机与背景）

- **应用背景**：大模型（LLM）服务通常以 MaaS（Model-as-a-Service）形式运行，云厂商在同一集群上托管数百乃至数千个模型（开源模型族、不同规模变体、用户微调模型）。MaaS 的两大目标是**最大化 goodput（满足 SLO 的请求数）**与**最小化实例数以提升硬件利用率**。
- **需求波动剧烈**：真实 trace（BurstGPT）显示请求率可在 2 秒内上涨 5×；LLM 自回归特性使 KVCache 占用在 3–12× 之间波动（Llama2-7B 服务 BurstGPT 需 190–760 GB KVCache）。因此**模型自动扩缩容（autoscaling）**成为必要机制：长期按平均负载配置实例，突发时从其他模型的空闲 GPU 上扩容。
- **核心矛盾（论文要解决的根本问题）**：
  - **扩缩容数据面太慢**：把 10–400 GB 参数加载到 GPU。GPU 服务器 SSD 每卡仅 2–10 Gbps，Llama3-8B 需 12.8 s；而 Llama3-8B 推理仅 80–900 ms，用户期望 <1 s 响应，要求扩缩容时间 <500 ms。
  - **stop-the-world 语义**：现有系统必须等全部参数加载完，新实例才能服务，扩缩容时间直接转化为排队延迟与 SLO 违约。
  - **主机缓存的取舍失效**：ServerlessLLM 等多级缓存方案依赖 CPU DRAM 命中（PCIe 256 Gbps），但实测命中率不足（miss 率 20–46%，与其论文 25–60% 一致）；根因是 MaaS 托管模型太多，无法在每台主机缓存所有模型，且**一次扩多个实例会涉及更多主机，miss 概率进一步上升**。
- **量化需求**：仿真表明，Qwen2.5-72B 要把 SLO 违约率控制在 60% 以下，需每卡 220 Gbps；要在 500 ms 内完成加载则需 576 Gbps/GPU，远超典型计算网络（200 Gbps/GPU）甚至主机 PCIe（256 Gbps）。**因此仅靠"更快加载"不够，必须让扩缩容"边加载边服务"（live）。**
- **整体含义**：论文主张把扩缩容数据面从"SSD/主机缓存 + 阻塞式加载"重构为"**计算网络组播 + 层粒度协同执行**"，从而在 **O(1)（甚至 0）主机缓存**的前提下同时获得快速与热（live）扩缩容，为 serverless 化模型服务提供可扩展的参数分发机制。

## 二、方法论

### 2.1 核心思想（两条主线）

1. **数据面变快：借道 GPU 间计算网络 + 网络优化组播，实现 O(1)/零缓存**
   - 观察一：GPU–GPU/CPU 计算网络（RDMA 100–400 Gbps、NVLink 1.6–3.6 Tbps）速度与主机缓存相当甚至更快。
   - 观察二：该网络在服务期**严重闲置**——即便在 PD 分离这类网络密集负载的峰值下，仍有 40% 以上带宽空闲。
   - 观察三：多实例扩容可用**串行转发链式组播**，加载时间与接收者数量无关，因此无需缓存；即使无已部署实例，也只需"聚合主机内存中每个模型一份副本"的 **O(1) 缓存**即可广播，理论上可消除所有 miss。
2. **数据面变活：把扩缩容抽象从实例级细化为层（layer）级，实现协同执行**
   - 打破"必须全部参数就绪才能出结果"的粗粒度抽象：新实例加载完第 1 层即可分担重载实例的该层计算，把激活值回传原实例继续后续层，**在参数未加载完时即提升系统吞吐**。
   - 举例：7 层模型，单实例吞吐 1/7；新实例加载 1 层后原实例只需算 6 层，吞吐升至 1/6；加载一半层后吞吐翻倍（仅用一半扩容时间）。

### 2.2 关键技术细节

- **全局参数池（Global Parameter Pool）**：集中记录每个模型参数在各 GPU（已部署实例）与各主机 CPU（本地缓存）上的位置；初始化时把每个模型一份副本均匀分散到主机 CPU，部署/回收时更新位置并回收/重载缓存，保证集群级至少一份可用副本。
- **网络建模**：将 NVLink 连接的 GPU 抽象为"逻辑实例组"（scale-up 域内带宽极高，可忽略），从而把 NVLink 从拓扑中消去；scale-out 采用 leaf–spine 简化模型（同 leaf 全互联，带宽取 min(BW_i, BW_j)；跨 leaf 不显式建模，依赖 VLT/ECMP 等上层协议）。
- **在线组播计划生成（贪心三阶段算法）**：
  1. **剪枝源**：剔除会与服务流量冲突的源，避免干扰（实测干扰会导致扩容时间 1.5×、尾 TBT 劣化 50%）；
  2. **目标分组**：把同一 scale-up 域（NVLink）的目标聚成组，组内用 NVLink AllGather 实现**并行分片传输**（源/目标各 4 GPU 时，每源 GPU 只发 1/4 分片，扩容时间降至 1/4）；
  3. **构造多条串行广播链** S→T1→T2→…→Tn：链式转发的传输时间与接收者数无关（带宽受限下的最优）；多条链可避免跨 leaf 慢链路，并在 PD 分离场景下让多个目标同时无干扰地 live 扩容。
  - **链序选择**：按链路带宽降序排列（先给带宽高的目标），使高带宽节点停机时间更短。
  - **利用 RDMA 双向性**：incast 与 outcast 流量互不干扰，因此可刻意从 decode 实例向 prefill 实例加载参数以规避同向冲突。
  - 该问题本质是异构网络上的组播规划（NP-hard），论文用"模型感知 + 服务静态数据流"特征的贪心法在线快速得到近优、无干扰解。
- **ZigZag 热调度（live autoscaling）**：
  - 三段式协议：①新实例开始加载即把排队/新请求重定向过去（请求负载远小于模型，重定向开销可忽略）；②首层加载完即开始执行首层，期间旧实例继续处理积压；③全部加载完后请求在两实例间均分。
  - **调度问题形式化**：给定 N 个等执行时间的请求批，决定流水线配置 (T_i, S_i)（目标/源实例各执行多少层），目标是最小化平均延迟 `Latency_avg = (Σ_req Σ_{i=1..req} S_i)/N`，约束为：
    - C1 流水线完整：S_i + T_i = L；
    - C2 流水线依赖：Σ_{j≤i} T_j ≤ Σ_{j<i} S_j（源执行第 i 个请求时目标必须已完成对应层）；
    - C3 加载限制：Time_l × T_i ≤ Σ_{j<i} T_j + (N−i+1)(T_i−1)（目标执行第 i 个请求的 T_i 层时这些层必须已加载，加载时间可与后续请求执行重叠）。
  - **朴素 best-effort 策略次优**：加载初期新实例能力有限，请求仍堆在旧实例上导致 SLO 违约（示例中请求 6 延迟 32 单位）。
  - **ILP-free 实现**：新实例维护可被旧实例 RPC 拉取的分布式优先队列（优先级 = FCFS 次序 + "下一层已加载"优先）；旧实例**仅在自己无积压时**才拉取执行，从而自然形成"延迟—回访"的 ZigZag 流水；未被执行完的请求保留在队列中，待更多层就绪后再次调度。ILP 对 Llama3-8B 求解 <40 ms，但对 80 层模型用无 ILP 版本彻底消除求解开销。
- **LLM 专门化优化**：
  - prefill 的层执行时间与批内 token 数近似线性，引入按 batch 画像的调节参数修正 ILP；
  - decode 批大小动态变化，但 ILP-free 调度天然适用；
  - 支持 PD colocation（PD 同置实例视为普通实例，ZigZag 同样适用）；
  - **PD 分离下的 decode 实例热扩容**：因参数加载与 KVCache 传输存在 incast 争用，直接热扩容不可行；改为先把部分 prefill 实例"变异"为 decode 实例，同时热扩容 prefill 实例补偿 prefill 吞吐；
  - **零成本预扩容 decode**：prefill 扩容需求可预示 decode 需求，故同时扩容 decode，把其扩容开销隐藏在 prefill 时间内（该策略对 ServerlessLLM 等基线同样生效，保证公平）。
- **控制面优化**：CUDA context 池 + 预加载 kernel（避免 ~500 ms context 创建）、C++/CUDA 原生实现避免 PyTorch `dlopen` 初始化、预建全互联 P2P 连接池（避免 NCCL 建组 ~100 ms）、故障时重新分发丢失的缓存副本以维持参数池不变式。

## 三、实验设计

- **系统实现**：BlitzScale 为 24,000 行 Rust/C++ 的 MaaS 系统，支持 PD 分离与 continuous batching，GPU kernel 来自 FlashInfer。
- **测试床（表 1）**：
  - Cluster A：4 主机 × 8× A800 80GB（共 32 GPU），NVLink 1.6 Tbps、RDMA 100 Gbps、Host–GPU PCIe 128 Gbps、SSD 10 Gbps；
  - Cluster B：2 主机 × 8× A100 80GB（共 16 GPU），PCIe 256 Gbps（无 NVLink）、RDMA 100 Gbps、SSD 10 Gbps。
- **Trace**：BurstGPT、AzureCode、AzureConv（Azure 生产 trace），用 TraceUpscaler 保持时间模式缩放，平均请求率约为集群最大服务能力的一半。
- **模型**：Llama3-8B（单卡）、Mistral-24B、Qwen2.5-72B（4 卡张量并行）。作者说明因系统只对模型规模敏感，故按规模简称。
- **对比基线**：
  1. **ServerlessLLM (S-LLM)**：SOTA 扩容系统，主机 DRAM 缓存 + TTL 淘汰，miss 时用满 SSD 带宽；
  2. **ServerlessLLM AllCache**：始终命中主机缓存的最优扩容版本（性能上界参照）；
  3. **DistServe (Full / Half)**：SOTA 非扩容 PD 分离系统，Full = 用满全部 GPU（过配置最优性能），Half = 按平均实例数配置；
  4. **vLLM**：PD colocation 场景对比（Full/Half）。
- **指标**：TTFT、TBT、SLO 违约率（5× 平均延迟，遵循 DistServe 传统定义）、GPU 时间（曲线积分）、主机缓存占用、网络占用、初始化/控制面耗时、消融。
- **公平性控制**：BlitzScale 与 S-LLM 各变体使用**同一扩缩容策略**；作者校准 DistServe 使其在关闭扩容时与 BlitzScale 性能一致；AllCache 作为"永远命中缓存"的乐观上界。

## 四、资源与算力

- **无训练过程**：本文为系统/推理服务论文，不涉及模型训练，因此没有训练时长、训练算力（GPU-hours）等数据。
- **推理评测算力**：
  - Cluster A：4 台主机 × 8 张 A800 80GB = **32 张 A800**（NVLink 互联）；
  - Cluster B：2 台主机 × 8 张 A100 80GB = **16 张 A100**（PCIe 互联）。
- 其他提及：致谢中提到阿里巴巴通义实验室在早期阶段提供测试床；硬件带宽调查（附录表 2）为云厂商公开配置的调研，非自建集群。
- 说明：论文未报告总 GPU 机时、单次实验时长、重复次数与随机种子等细节；单次端到端评估时间窗约 5 分钟（图 17 横轴 00:00–05:00）。

## 五、实验数量与充分性

- **实验组数概览（约 9 类、20+ 图/表）**：
  1. **动机刻画实验**：图 1（请求率/算力/内存需求时间线）、图 3(a)–(d)（不同扩容停顿时长 × 4 种模型/指标下的 SLO 违约曲线）、图 3(e)(f)（DistServe 网络占用）、图 4（ServerlessLLM 缓存 miss 率）、图 8（网络干扰对扩容速度与 TBT 的影响）。
  2. **端到端主实验**：图 17，3 组 (trace × 模型 × 集群) 组合（BurstGPT×72B×A、AzureCode×8B×B、AzureConv×24B×A），每组含请求率、TTFT、TBT 时序及两者 CDF。
  3. **与非扩容系统对比**：图 18（AzureConv×24B，DistServe Full/Half vs S-LLM vs BlitzScale 的延迟 CDF + GPU 时间）。
  4. **主机缓存占用对比**：图 19（3 个 workload，S-LLM vs
