---
title: "Enabling Efficient {GPU} Communication over Multiple {NICs} with {FuseLink}"
title_zh: FuseLink：利用多网卡实现高效GPU通信
authors: "Zhenghang Ren, Yuxuan Li, Zilong Wang, Xinyang Huang, Wenxue Li, Kaiqiang Xu, Xudong Liao, Yijun Sun, Bowen Liu, Han Tian, Junxue Zhang, Mingfei Wang, Zhizhen Zhong, Guyue Liu, Ying Zhang, Kai Chen"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-ren.pdf"
tags: ["query:dcn-comm"]
score: 8.0
evidence: 面向多网卡与不均衡全对全流量的高效GPU通信
tldr: 机器学习集群为提升跨服务器GPU通信带宽而在单机堆叠多块网卡，但静态的GPU与网卡绑定导致在专家并行全对全等不均衡流量下出现热点网卡瓶颈。本文提出FuseLink，通过动态扩展跨网卡的服务器间网络连接，实现通信流在多网卡间的灵活分摊。实验表明该设计能充分利用全部网卡带宽、缓解热点拥塞，提升集合通信与训练任务的整体吞吐。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
motivation: 现有系统因静态GPU-网卡绑定，在ML任务的不均衡通信下无法充分利用多网卡带宽。
method: FuseLink动态扩展跨网卡的服务器间网络，将GPU通信流灵活分摊到多块网卡上。
result: 实验显示FuseLink缓解热点网卡瓶颈，充分利用全部网卡带宽并提升通信吞吐。
conclusion: 该工作为多网卡集群中的高效GPU集合通信提供了可行的系统设计。
---

## Abstract
Machine learning (ML) clusters stack multiple network interface cards (NICs) within each server to improve inter-server GPU communication bandwidth. However, existing systems fall short in fully utilizing NICs because of static GPU-NIC bindings. This leads to bottlenecks at hot-spot NICs when handling imbalanced communication in ML tasks. For example, large language model serving instances may have different communication demands across NICs; expert-parallel training tasks have imbalanced all-to-all traffic; and the embedding transmission volumes during recommendation model training vary across GPUs. To fully utilize all NICs, we propose FuseLink to enable efficient GPU communication over multiple NICs. FuseLink extends inter-server network by integrating high-speed intra-server connections, and leverages GPUs to efficiently relay traffic to idle NICs. We implement FuseLink and integrate it into NCCL, so that ML applications can benefit from FuseLink seamlessly without code modifications. Compared to NCCL, we demonstrate that FuseLink achieves up to 212GBps bandwidth between two inter-server GPUs and accelerates ML tasks with dynamic traffic patterns. Specifically, it reduces the latencies of first-token generation in LLM model servings by 1.04-2.73×, improves the training throughput of mixture-of-experts model by up to 1.3×, and accelerates deep learning recommendation model training by up to 1.2×.

---

## 论文详细总结（自动生成）

### 1. 核心问题与整体含义

- **研究背景**：分布式机器学习对 GPU 间通信带宽要求极高。服务器内已有 NVLink/NVSwitch 等 Tbps 级高速互连，但跨服务器 RDMA 网卡带宽增长相对较慢，成为主要瓶颈。为此，ML 集群通常在单机内堆叠多块 RDMA NIC，并让每块 NIC 与特定 GPU 通过 PCIe 静态绑定。
- **核心问题**：静态 GPU–NIC 绑定只适合流量均衡的任务。对于 disaggregated LLM serving、专家并行 MoE 训练、DLRM 训练等动态、不均衡流量场景，部分 NIC 会成为热点瓶颈，而其他 NIC 空闲，导致多 NIC 资源无法充分利用。
- **整体含义**：论文提出 **FuseLink**，将服务器内高速 GPU 互连（如 NVLink）作为跨服务器网络的延伸，通过 GPU 中继把流量动态调度到空闲 NIC，从而把多块 NIC“融合”为一条高带宽链路。目标是在不修改 ML 应用代码、不调整并行策略的前提下，提高跨服务器 GPU 通信带宽和 NIC 利用率。

### 2. 方法论：核心思想与关键技术

- **核心思想**：
  - 打破 GPU 与 NIC 的静态绑定，将服务器内、服务器间网络资源统一抽象。
  - 当某 GPU 的直连 NIC 繁忙时，可经 NVLink 将数据中继到其他 GPU，再由其直连或间接 NIC 发送。
  - 通过运行时监控 NIC 负载，动态选择直连 NIC 或空闲间接 NIC，实现多 NIC 带宽聚合。

- **关键技术细节**：
  - **基于内存重映射的高效中继**：利用统一虚拟地址和 GPU 内存重映射，将网络缓冲区从原 GPU 映射到中继 GPU。应用仍向原虚拟地址写数据，实际写入中继 GPU 内存，避免额外显式拷贝和设备同步。论文比较了 D1–D4 四种中继方式，选择 D1（重映射网络缓冲区）以获得最高间接 NIC 吞吐和较低延迟。
  - **连接建立与路径选择**：FuseLink 探索服务器内 PCIe/NVLink 拓扑，识别每个 GPU 的直连 NIC 和间接 NIC；若 NIC 与 GPU 直连则走直连路径，否则选择合适的中继 GPU。
  - **Credit 与负载感知调度**：接收端在 credit 中携带接收侧空闲 NIC 信息；发送端同时监控自身 NIC 负载。选择顺序为：优先使用空闲的直连 NIC；否则选择空闲的间接 NIC；若均不可用则回退到直连 NIC。对间接 NIC 的未完成操作数量进行限制，以降低竞争风险。
  - **接收端路由**：数据可能先到达与所选 NIC 直连的中继 GPU，再通过内存重映射转发到目标 GPU；发送端也可能在首次发送时走次优路径，随后重映射中继缓冲区以优化后续传输。
  - **无中断中继**：通过高优先级 worker 的网络完成情况判断 NIC 忙闲；高优先级 worker 对其直连 NIC 具有最高优先级。TP 通信和中继期间将 NIC 标记为忙，避免干扰。中继内存设置可配置上限，并在内存压力下优先释放中继内存，降低 OOM 风险。
  - **低开销控制面**：批量检查 NIC 状态，允许有界竞争和有界次优路径；利用 ML 消息大、NIC 状态变化相对缓慢的特点降低调度开销。
  - **NCCL 集成**：以约 3000 行 C++ 实现为独立网络模块，拦截 NCCL proxy thread 的连接建立、缓冲区注册和收发调用，替换默认 InfiniBand 网络层，使应用无需改代码即可使用。

### 3. 实验设计

- **测试平台**：
  - 服务器配置 Intel 8480C CPU、8 块 NVIDIA Hopper GPU、8 块 ConnectX-7 400Gbps NIC。
  - 服务器内 GPU 通过八 lane NVLink/NVSwitch 连接，理论带宽约 200GB/s；每块 NIC 通过 PCIe bridge 与一块 GPU 直连。
  - FuseLink 调度器以守护进程运行，ML worker 以 NCCL 插件方式加载 FuseLink。

- **对比基线**：
  - 主要基线为 **NCCL + PXN** 或静态 NIC 绑定/仅使用直连 NIC 的 NCCL 配置。
  - 微基准中还比较了不同中继方式 D1–D4、流量喷洒与 FuseLink 的中断避免机制，以及 Table 2 中逐步加入设计模块的消融结果。

- **微基准**：
  - 以 512KB 消息测试跨服务器 GPU 带宽，改变 FuseLink 可使用的最大 NIC 数量，观察带宽扩展性。
  - 测试调度开销，包括 NIC 监控、NIC 选择、发送/接收处理、缓冲区重映射和切换 NIC 的开销。
  - 用长短流场景比较直接 NIC 通信、流量喷洒和 FuseLink 的中断避免效果。

- **端到端场景**：
  - **Disaggregated LLM serving**：OPT-30B，prefill/decode 分离；测试 8 实例 TP=1、4 实例 TP=2、2 实例 TP=4；请求来自 Azure 公共 trace，Poisson 到达；指标为 TTFT，并报告 P50/P99。
  - **专家并行 MoE 训练**：Mixtral 8×22B，TP=4、EP=8，每台服务器并行训练两个 expert；比较训练迭代时间和吞吐。
  - **DLRM 训练**：Criteo 广告数据集，32 个 GPU worker，专用 embedding server，batch size=1024，测试不同 embedding cache 大小下的迭代时间。

### 4. 资源与算力

- 论文明确给出的算力信息：
  - 单机测试平台为 **8 块 NVIDIA Hopper GPU + 8 块 400Gbps ConnectX-7 NIC**。
  - DLRM 端到端实验使用 **32 个 GPU worker**。
  - LLM serving 使用 OPT-30B；MoE 训练使用 Mixtral 8×22B。
- **未明确说明**：
  - 实验总共使用多少台服务器、多少块 GPU。
  - 端到端训练/服务实验的总训练时长、GPU-hours、能耗或成本。
  - 因此无法据此精确评估大规模部署成本和能效。

### 5. 实验数量与充分性

- **实验数量**：
  - 包含多组微基准：多 NIC 带宽扩展、调度开销分解、长短流干扰实验。
  - 包含设计消融：Table 2 逐步评估高效中继、消除中断、降低 NIC 竞争、高效调度等模块的贡献。
  - 包含三类端到端负载：LLM serving 的 3 种 TP/实例配置及 P50/P99 表、MoE 训练迭代、DLRM 多种 cache 大小。
  - 总体覆盖了论文声称的主要动态流量场景，实验数量对系统论文而言较充分。
- **充分性与客观性**：
  - 优点是与 NCCL/PXN 等实际系统对比，使用公共 trace 和标准数据集，报告了 P50/P99 和消融结果。
  - 局限是端到端实验规模仍偏原型验证，主要集中在一类 NVIDIA Hopper + ConnectX-7 硬件；未与 MP-RDMA、ARK、NetChannel 等多路径或数据路径优化系统进行端到端比较；未系统评估多服务器规模、异构 GPU–NIC 比例、故障和租户隔离等。实验未报告误差棒或多次运行方差，统计严谨性有限。

### 6. 主要结论与发现

- FuseLink 可把跨服务器双 GPU 通信带宽提升至 **212GBps**，显著高于单 NIC 约 50GBps 的默认配置，并超过八 lane NVLink 的理论带宽上限。
- 在动态、不均衡流量任务中：
  - LLM serving 的 TTFT 降低 **1.04–2.73×**；
  - MoE 训练吞吐提升最高 **1.3×**；
  - DLRM 训练加速最高 **1.2×**。
- 通过动态利用空闲 NIC 和 NVLink 中继，FuseLink 能缓解热点 NIC 瓶颈，提高整体 NIC 利用率，同时通过优先级和内存管理避免明显干扰对等 GPU 的任务。
- 消融表明，高效中继、无中断机制、NIC 竞争缓解和高效调度均对最终带宽有贡献；其中多 NIC 聚合和调度优化带来主要增益。

### 7. 优点

- **问题定位清晰**：准确指出静态 GPU–NIC 绑定在动态不均衡流量下的根本缺陷。
- **方法具有系统创新性**：把服务器内高速互连纳入跨服务器数据路径，并通过内存重映射减少拷贝和同步，设计巧妙。
- **工程可部署性强**：以 NCCL 插件形式集成，应用无需修改代码，具备实际部署价值。
- **兼顾性能与隔离**：不仅追求多 NIC 聚合，还考虑对等 GPU 通信干扰、NIC 竞争和中继内存 OOM 风险。
- **实验较完整**：微基准、消融和三类真实 ML 负载结合，能够支撑核心论点。

### 8. 不足与局限

- **适用场景有限**：最适合点对点、不均衡流量；对流量均匀的集合通信（如 ring all-reduce）收益有限，甚至可能需要额外 worker 放置调整。
- **依赖特定硬件与系统能力**：依赖 NVLink 类高速服务器内互连、统一虚拟地址、GPU P2P 和 RDMA 内存重映射；在其他互连或非 NVIDIA 平台上的可移植性仍需验证。
- **资源开销与风险**：中继会占用中继 GPU 的内存和带宽；OOM 规避是 best-effort

，无法提供硬性隔离保证；在极端内存压力或多任务并发下，中继仍可能对宿主 GPU 上的计算或通信造成资源争用。此外，中继会消耗 NVLink/NVSwitch 带宽和 GPU 内存带宽，因此其收益依赖于服务器内互连带宽显著高于跨服务器 NIC 带宽这一前提；若服务器内互连本身成为瓶颈，FuseLink 的扩展空间会受限。

- **控制面存在有界竞争与次优路径**：FuseLink 通过批量检查、允许有界竞争和首次发送可能走次优路径来降低调度开销，这意味着它并非全局最优调度器。在高并发、NIC 状态快速变化或消息较小时，credit 信息的滞后可能降低路径选择质量，甚至引入额外中继跳数。
- **对 NCCL 内部实现的依赖较强**：约 3000 行 C++ 的 NCCL 网络插件通过拦截 proxy thread 的连接建立、缓冲区注册和收发调用来实现，因而与 NCCL 版本、InfiniBand 网络层接口及 GPU 内存重映射能力耦合较深。NCCL 升级或非 NCCL 通信库场景下的兼容性和维护成本需要进一步评估。
- **实验统计与规模覆盖仍不完整**：端到端结果主要基于单类 Hopper + ConnectX-7 平台和原型规模，未系统报告多次运行方差、置信区间或误差棒；也未充分覆盖多服务器规模扩展、异构 GPU–NIC 配比、链路故障、拥塞突发、租户隔离以及与非 NVIDIA 互连的对比。因此，论文结论在更大规模生产集群中的外推仍需谨慎。
- **与相关工作的端到端比较不足**：论文主要与 NCCL/PXN 或静态绑定基线比较，虽能说明核心收益，但尚未与 MP-RDMA、ARK、NetChannel 等多路径/数据路径优化系统进行端到端对照，难以完全界定 FuseLink 相对于已有多 NIC 聚合方案的综合优势边界。

### 9. 总体评价与启示

- **总体定位**：FuseLink 是一篇问题导向明确、系统实现完整的网络/ML 系统交叉论文。其核心贡献不是单纯发明新的 RDMA 协议，而是重新审视“GPU–NIC 静态绑定”这一常见部署假设，将服务器内高速互连转化为跨服务器网络的动态扩展资源，并通过内存重映射、credit 调度和 NCCL 插件化实现较低侵入的部署。
- **方法价值**：论文将“多 NIC 聚合”从链路层/传输层问题拓展到“GPU 内存语义 + 服务器内互连 + 跨服务器网络”的协同设计层面。基于内存重映射的中继避免了显式拷贝和同步，是该方法能够低开销运行的关键；负载感知与无中断中继则体现了对 ML 通信干扰和资源隔离的实际考虑。
- **实践启示**：对于存在动态、不均衡跨服务器流量的 ML 集群，单纯增加 NIC 数量而不改变静态绑定策略可能无法充分利用带宽。系统设计者可将 NIC 视为服务器级共享资源，而非 GPU 的固定附属设备；但同时也需配套内存上限、优先级和故障回退机制，以避免中继引入新的尾延迟和资源争用。
- **研究空间**：未来工作可进一步探索全局最优的多跳/多路径调度、跨租户隔离与 QoS、异构互连下的可移植实现、与集合通信算法的协同优化，以及在真实大规模生产集群中的长期稳定性与成本收益评估。

（完）
