---
title: "{ACCL+}: an {FPGA-Based} Collective Engine for Distributed Applications"
title_zh: ACCL+：面向分布式应用的基于FPGA的集合通信引擎
authors: "Zhenhao He, Dario Korolija, Yu Zhu, Benjamin Ramhorst, Tristan Laan, Lucian Petrica, Michaela Blott, Gustavo Alonso"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi24-he.pdf"
tags: ["query:dcn-comm"]
score: 9.0
evidence: 基于FPGA的集合通信库，支持RDMA与集合卸载
tldr: 针对FPGA分布式应用缺乏可移植集合通信支持的问题，本文提出开源集合通信库ACCL+，可在不同平台支持UDP、TCP和RDMA，实现FPGA间直接集合通信并可作为CPU应用的集合卸载引擎。在100Gb/s FPGA集群上的评估表明其能有效完成集合操作并降低CPU网络负担。该工作为异构加速器集群的集合通信设计提供了可移植且可扩展新集合的实现基础。
source: OSDI-2024-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-001.webp\", \"caption\": \"\", \"page\": 1, \"index\": 1, \"width\": 875, \"height\": 356}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-002.webp\", \"caption\": \"\", \"page\": 1, \"index\": 2, \"width\": 875, \"height\": 360}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-003.webp\", \"caption\": \"\", \"page\": 2, \"index\": 3, \"width\": 1224, \"height\": 650}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-004.webp\", \"caption\": \"\", \"page\": 2, \"index\": 4, \"width\": 1223, \"height\": 651}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-005.webp\", \"caption\": \"\", \"page\": 5, \"index\": 5, \"width\": 1671, \"height\": 1431}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-006.webp\", \"caption\": \"\", \"page\": 6, \"index\": 6, \"width\": 1776, \"height\": 1678}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-007.webp\", \"caption\": \"\", \"page\": 6, \"index\": 7, \"width\": 2004, \"height\": 937}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-008.webp\", \"caption\": \"\", \"page\": 7, \"index\": 8, \"width\": 1304, \"height\": 963}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-009.webp\", \"caption\": \"\", \"page\": 7, \"index\": 9, \"width\": 1315, \"height\": 974}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-010.webp\", \"caption\": \"\", \"page\": 8, \"index\": 10, \"width\": 2272, \"height\": 1308}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-011.webp\", \"caption\": \"\", \"page\": 9, \"index\": 11, \"width\": 1617, \"height\": 882}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-012.webp\", \"caption\": \"\", \"page\": 10, \"index\": 12, \"width\": 900, \"height\": 400}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-013.webp\", \"caption\": \"\", \"page\": 10, \"index\": 13, \"width\": 890, \"height\": 190}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-014.webp\", \"caption\": \"\", \"page\": 10, \"index\": 14, \"width\": 889, \"height\": 423}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-015.webp\", \"caption\": \"\", \"page\": 11, \"index\": 15, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-016.webp\", \"caption\": \"\", \"page\": 11, \"index\": 16, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-017.webp\", \"caption\": \"\", \"page\": 11, \"index\": 17, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-018.webp\", \"caption\": \"\", \"page\": 11, \"index\": 18, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-019.webp\", \"caption\": \"\", \"page\": 11, \"index\": 19, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-020.webp\", \"caption\": \"\", \"page\": 11, \"index\": 20, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-021.webp\", \"caption\": \"\", \"page\": 11, \"index\": 21, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-022.webp\", \"caption\": \"\", \"page\": 11, \"index\": 22, \"width\": 640, \"height\": 480}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-023.webp\", \"caption\": \"\", \"page\": 11, \"index\": 23, \"width\": 2112, \"height\": 1584}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-024.webp\", \"caption\": \"\", \"page\": 11, \"index\": 24, \"width\": 2112, \"height\": 1585}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-025.webp\", \"caption\": \"\", \"page\": 12, \"index\": 25, \"width\": 630, \"height\": 469}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-026.webp\", \"caption\": \"\", \"page\": 12, \"index\": 26, \"width\": 630, \"height\": 470}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-027.webp\", \"caption\": \"\", \"page\": 12, \"index\": 27, \"width\": 1740, \"height\": 386}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-028.webp\", \"caption\": \"\", \"page\": 13, \"index\": 28, \"width\": 2254, \"height\": 1020}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-029.webp\", \"caption\": \"\", \"page\": 13, \"index\": 29, \"width\": 1305, \"height\": 627}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-030.webp\", \"caption\": \"\", \"page\": 14, \"index\": 30, \"width\": 700, \"height\": 600}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-accl-an-fpga-based-collective-engine-for-distributed-applications/fig-031.webp\", \"caption\": \"\", \"page\": 14, \"index\": 31, \"width\": 2310, \"height\": 1981}]"
motivation: FPGA在云数据中心日益普及，但分布式FPGA应用缺乏可移植、可扩展的集合通信支持。
method: 提出开源FPGA集合通信库ACCL+，支持UDP、TCP和RDMA，并提供FPGA间直接集合通信与CPU集合卸载能力。
result: 在100Gb/s FPGA集群上评估，ACCL+能够执行集合通信并减轻CPU网络处理负担，且支持用户扩展新集合而无需整体重综合。
conclusion: ACCL+为FPGA集群和SmartNIC场景提供了可移植的集合通信基础设施，有助于推动分布式加速器应用。
---

## Abstract
FPGAs are increasingly prevalent in cloud deployments, serving as Smart-NICs or network-attached accelerators. To facilitate the development of distributed applications with FPGAs, in this paper we propose ACCL+, an open-source, FPGA-based collective communication library. Portable across different platforms and supporting UDP, TCP, as well as RDMA, ACCL+ empowers FPGA applications to initiate direct FPGA-to-FPGA collective communication. Additionally, it can serve as a collective offload engine for CPU applications, freeing the CPU from networking tasks. It is user-extensible, allowing new collectives to be implemented and deployed without having to re-synthesize the entire design. We evaluated ACCL+ on an FPGA cluster with 100 Gb/s networking, comparing its performance against software MPI over RDMA. The results demonstrate ACCL+'s significant advantages for FPGA-based distributed applications and its competitive performance for CPU applications. We showcase ACCL+'s dual role with two use cases: as a collective offload engine to distribute CPU-based vector-matrix multiplication, and as a component in designing fully FPGA-based distributed deep-learning recommendation inference.

---

## 论文详细总结（自动生成）

# ACCL+ 论文结构化总结

## 1. 核心问题与整体含义（研究动机与背景）

- **背景**：FPGA 正越来越多地部署在云数据中心，作为 SmartNIC、网络附属加速器或解耦加速器使用；部分 FPGA 可直接连接 100 Gb/s 网络。
- **核心问题**：FPGA 分布式应用缺少类似软件生态中 MPI/NCCL 的高层**集合通信抽象**。现有方案要么依赖 CPU 转发通信，增加延迟；要么只支持底层链路协议，难以扩展到数据中心分组交换网络；要么把集合模块静态固化在硬件中，修改需长时间重新综合。
- **具体挑战**：
  - **多样化传输协议支持**：需兼容 UDP、TCP、RDMA，并能与 CPU、加速器混合部署互操作。
  - **集合实现灵活性**：需在运行时选择不同集合与算法，且修改集合实现不必重新综合整个设计。
  - **跨应用与平台可移植性**：需支持分区内存模型与共享虚拟内存模型，以及消息传递和流式两种通信模型。
- **整体含义**：论文提出开源 FPGA 集合通信库 **ACCL+**，既能让 FPGA 应用直接发起 FPGA-to-FPGA 集合通信，也可作为 CPU 应用的集合卸载引擎，目标是成为异构分布式加速器集群中的可移植集合通信基础设施。

## 2. 方法论：核心思想与关键技术

### 2.1 核心思想
- 采用**模块化、分层架构**，将平台相关 IO 管理、运行时、网络协议适配与集合逻辑解耦。
- 核心是 **CCLO（CCL Offload Engine）**：将集合控制面与并行数据面分离，实现“灵活控制 + 高性能数据通路”。
- 提供两套 API：
  - **MPI-like 集合 API**：面向消息传递，操作内存缓冲区，便于移植现有 MPI 应用。
  - **Streaming 集合 API**：数据直接来自/流向 FPGA kernel 的流接口，避免内存缓冲，支持通信与计算重叠。

### 2.2 关键技术细节
- **CCLO 控制面**：
  - 以嵌入式微控制器（uC）为中心，用固件实现集合算法与同步协议，修改集合无需重新综合硬件。
  - uC 通过高层**数据移动原语**驱动硬件数据移动处理器（DMP）；每条原语含两个操作数槽和一个结果槽，适配 reduce 等二元操作。
  - **RxBuf Manager（RBM）**：自主管理接收缓冲、重组网络包、维护消息状态表，减少 uC 负担。
- **CCLO 数据面**：
  - **DMP** 隐藏内存访问延迟，将微码译码后分发给多个计算单元，流水化执行内存读取、网络发送、接收检查等。
  - Tx/Rx 系统具有 **512-bit 数据通路**，负责消息签名打包/解包；签名包含消息类型、目的 rank、长度、tag、序列号等。
  - 内部采用 **Network-on-Chip** 按包路由；支持可编译期启用的**流式插件**，用于 sum/max 等归约，或压缩/加密等一元操作。
- **消息同步协议**：
  - **Eager**：发送端立即发送，接收端暂存到 Rx buffer；适合小消息、低延迟，基于 UDP/TCP 或双边 RDMA。
  - **Rendezvous**：先解析目标地址，再用单边 RDMA WRITE 直接写入目的端；发送端随后发送小控制消息通知接收端 uC，避免轮询。
- **集合算法**：
  - Broadcast：小规模 one-to-all，大规模递归倍增。
  - Reduce/Gather：小消息 all-to-one，大消息二叉树，避免 incast。
  - All-to-all：线性算法。
  - 算法可在运行时通过配置参数调整。
- **平台适配**：
  - 支持 **AMD Vitis**（分区内存模型，XRT/XDMA/Data Mover）与 **Coyote**（共享虚拟内存、TLB、RDMA 服务）。
  - 提供 **ZMQ 仿真平台**，支持功能仿真与周期精确仿真。
  - 通过 `BaseBuffer`、`BaseDevice` 等基类及平台子类实现可移植性。
- **协议卸载引擎（POE）**：
  - 支持 100 Gb/s UDP、TCP 以及 Coyote 的 RDMA 服务。
  - CCLO 使用与 POE 无关的内部 meta/data 流接口，通过适配器对接不同协议。
  - RDMA 场景下 CCLO 作为内存管理与 RDMA POE 之间的“bump-in-the-wire”引擎。

## 3. 实验设计

- **测试平台**：
  - 10 台 AMD EPYC CPU + 10 块 **AMD Alveo-U55C FPGA**，每 CPU 配 100 Gb/s Mellanox NIC，每 FPGA 配 100 Gb/s 以太网接口，统一接入 Cisco Nexus 9336C-FX2 交换机。
  - 主要使用 **Coyote + RDMA** 配置；部分实验使用 **Vitis XRT + TCP**。
- **对比基线**：
  - **MPICH 4.0.2 over TCP**。
  - **OpenMPI 4.1.3 + OpenUCX 1.13.1 over RDMA**，运行于 CPU + Mellanox 100 Gb/s NIC。
  - 部分实验对比早期 **ACCL**（同样基于嵌入式微控制器、TCP/XRT）。
- **评估场景**：
  - **F2F**：FPGA 应用数据直接经 ACCL+ 网络传输，对比“先搬到 CPU 内存再用软件 MPI”的建模路径。
  - **H2H**：CPU 应用数据经 FPGA 卸载传输，对比 CPU 直接使用软件 MPI/NIC。
- **微基准**：
  - Send/Recv 吞吐。
  - CCLO 调用延迟（NOP 操作）。
  - Broadcast、Gather、Reduce、All-to-all 的延迟随消息大小变化。
  - Reduce 的延迟随 rank 数扩展（8 KB、128 KB）。
  - TCP/XRT 平台下 Gather、Reduce 与软件 MPI TCP、ACCL 的对比。
  - 每项微基准平均 **250 次运行**。
- **用例评估**：
  - **CPU 分布式向量-矩阵乘法**：按列划分权重矩阵，用 ACCL+ RDMA 或 MPI RDMA 做 reduce，对比单节点执行，使用 Eigen。
  - **全 FPGA 分布式 DLRM 推理**：工业级推荐模型，嵌入表约 50 GB，部署在 10 块 U55C 上；FC1 用棋盘块分解到 8 块 FPGA，FC2/FC3 各 1 块；使用 TCP/XRT 后端、32-bit 定点、115 MHz；对比 Intel Xeon Platinum 8259CL（32 vCPU）+ TensorFlow Serving。
- **资源评估**：报告 CCLO、TCP POE、RDMA POE 及 DLRM 各层的 CLB、LUT、DSP、BRAM、URAM 占用。

## 4. 资源与算力

- 论文**未涉及 GPU 型号、GPU 数量或模型训练时长**；该工作不是训练类研究，无 GPU 算力消耗报告。
- 实验算力主要为：
  - **10 块 AMD Alveo-U55C FPGA**；
  - **10 颗 AMD EPYC CPU**；
  - 100 Gb/s Mellanox NIC 与 100 Gb/s FPGA 以太网接口；
  - Cisco Nexus 9336C-FX2 交换机。
- 运行频率：微基准中 ACCL+ 约 **250 MHz**；DLRM 用例因设计复杂度为 **115 MHz**。
- 论文未报告总实验机时、功耗或成本。

## 5. 实验数量与充分性

- **实验组数**：覆盖较广，包括：
  - Send/Recv 吞吐；
  - 调用延迟；
  - 4 类集合操作 × 多消息大小；
  - rank 扩展实验；
  - 两种同步协议对比；
  - TCP/XRT 与 RDMA/Coyote 两种平台；
  - 2 个完整用例；
  - 资源利用率分析。
- **充分性**：
  - 微基准较系统，覆盖吞吐、延迟、扩展性和平台差异；每点 250 次平均，统计上较稳健。
  - 用例具有代表性：一个验证 CPU 集合卸载，一个验证全 FPGA 分布式推理。
- **客观性与公平性**：
  - 与主流软件 MPI（MPICH、OpenMPI/UCX）对比合理，基线配置较明确。
  - 但 F2F 场景中软件 MPI 的端到端时间部分是**建模分解**（PCIe 搬运 + MPI + 搬回 + kernel 调用），并非完全实测的端到端应用，存在一定估算偏差。
  - H2H 场景中 ACCL+ 在 Vitis 上需额外 staging 拷贝，XRT 调用延迟也较高，论文对此有说明。
  - 扩展性实验最多到 8 ranks，集群规模有限；DLRM 用到 10 块 FPGA，但集合扩展性未在更大规模上验证。
  - 未与 GPU NCCL/RCCL 或 DPU 集合卸载方案做直接实验对比，跨加速器生态的公平性有限。

## 6. 主要结论与发现

- ACCL+ 的 Send/Recv 峰值吞吐达 **95 Gb/s**，接近饱和 100 Gb/s 网络。
- **F2F 场景**：ACCL+ 相比“FPGA 数据经 CPU 软件 MPI”显著降低延迟，避免 PCIe 往返和 CPU 参与。
- **H2H 场景**：ACCL+ 性能与软件 RDMA MPI 具有竞争力；Broadcast、Gather 常优于软件 MPI，Reduce、All-to-all 仅小幅受益甚至略逊，主要因软件 MPI 算法调优更细。
- 集合卸载可释放 CPU 周期、降低 CPU cache 压力。
- Eager 协议在小消息下可因免握手而优于 Rendezvous；大消息下 Rendezvous 与树形算法更合适。
- TCP/XRT 平台上 ACCL+ 优于软件 MPI TCP 和早期 ACCL，说明将更多任务卸载到硬件数据面有效。
- **CPU 矩阵乘用例**：使用 ACCL+ 做 reduce 通常比 MPI 获得更低执行时间，部分配置出现超线性扩展，归因于分区后权重矩阵可放入 L2/L3 cache。
- **DLRM 用例**：ACCL+ 支持在 10 块 FPGA 上分布式部署工业级推荐模型，推理延迟比 CPU 基线低**两个数量级以上**，吞吐高**一个数量级以上**。
- 资源上，POE（尤其 TCP POE）占用大部分资源，CCLO 本身资源开销相对较小。

## 7. 优点

- **架构设计清晰**：控制面/数据面解耦，兼顾灵活性与高性能；集合算法可用固件修改而无需重新综合。
- **可移植性强**：同时支持 Vitis 分区内存与 Coyote 共享虚拟内存，支持 UDP/TCP/RDMA，并提供仿真平台。
- **双角色定位有实用价值**：既服务 FPGA 原生应用，又可作为 CPU 集合卸载引擎，拓展了 SmartNIC 场景。
- **接口设计完整**：MPI-like 与 streaming 两套 API 覆盖消息传递和流式应用，streaming 接口可绕过内存层级降低延迟。
- **实验较系统**：微基准、扩展性、协议对比、平台对比和两个真实用例结合，结论有较强支撑。
- **工程贡献明确**：开源实现、平台适配、仿真与调试支持，对后续 FPGA 集合通信研究有基础设施价值。

## 8. 不足与局限

- **规模有限**：集合扩展性实验最多 8 ranks，DLRM 为 10 块 FPGA；更大规模下的可扩展性、拥塞和容错行为未充分验证。
- **算法调优不如软件 MPI 细粒度**：H2H 下 Reduce、All-to-all 未稳定优于 MPI；论文也承认当前仅支持有限算法选项。
- **F2F 基线部分为建模**：软件 MPI 的 FPGA-to-FPGA 路径由分步测量组合估算，可能引入偏差。
- **平台差异影响结论**：Coyote 统一内存使 F2F/H2H 差异很小；Vitis 上 host 数据需 staging 拷贝，ACCL+ 对 host 应用有额外开销。
- **协议与网络条件简化**：仿真平台使用 ZMQ，不模拟丢包和重排序；真实数据中心中的拥塞、多租户干扰未评估。
- **缺少跨加速器对比**：未与 GPU NCCL/RCCL、DPU 集合卸载等方案直接比较，难以判断在异构集群中的整体竞争力。
- **资源开销**：TCP POE 资源占用较高；DLRM 用例中 URAM、DSP 成为瓶颈，资源受限场景下部署成本需考虑。
- **能效与成本未评估**：未报告功耗、性价比或总拥有成本，实际部署决策信息不足。
- **应用限制**：ACCL+ 主要面向集合通信和流式数据移动；对复杂非规则通信、动态拓扑或大规模故障恢复的支持未展开。

（完）
