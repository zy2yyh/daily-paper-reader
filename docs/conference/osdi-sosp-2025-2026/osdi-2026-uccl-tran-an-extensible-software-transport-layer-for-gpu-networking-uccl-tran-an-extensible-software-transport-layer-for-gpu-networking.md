---
title: "{UCCL-Tran}: An Extensible Software Transport Layer for {GPU} Networking"
title_zh: UCCL-Tran：面向GPU网络的可扩展软件传输层
authors: "Yang Zhou, Zhongjie Chen, Ziming Mao, ChonLam Lao, Shuo Yang, Pravein Govindan Kannan, Xizhi Zhang, Jiaqi Gao, Yilong Zhao, Yongji Wu, Kaichao You, Fengyuan Ren, Zhiying Xu, Costin Raiciu, Ion Stoica"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-zhou-yang.pdf"
tags: ["query:dcn-comm"]
score: 9.0
evidence: 可扩展的软件RDMA传输与多路径机制提升ML集合通信性能
tldr: RDMA网卡上的主机网络传输难以演进，单路径RDMA流量易发生流冲突，严重削弱ML集合通信性能。UCCL-Tran将现有RDMA网卡的数据路径与控制路径解耦，把可扩展的传输控制逻辑高效运行在主机CPU上，从而支持多路径传输等硬件难以实现的创新。基于该传输的ML集合通信性能最高提升4.5倍。该工作为GPU网络提供了可演进的软件传输层，对低延迟数据中心通信与集合通信优化具有直接价值。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: ML负载对网络要求快速演进，但RDMA网卡的主机传输难以修改，单路径流量易冲突拖慢集合通信。
method: UCCL-Tran解耦RDMA网卡数据路径与控制路径，在主机CPU上运行可扩展的软件传输控制，实现多路径传输。
result: 在UCCL-Tran之上运行ML集合通信，吞吐相比原方案最高提升4.5倍。
conclusion: 该工作表明软件可扩展传输能有效缓解流冲突，为GPU网络通信优化提供了新路径。
---

## Abstract
Fast-evolving machine learning (ML) workloads have increasing requirements for networking. However, host network transport on RDMA NICs is hard to evolve, causing problems for ML workloads. For example, single-path RDMA traffic is prone to flow collisions that severely degrade collective communication performance. We present UCCL-Tran, an extensible software transport layer to evolve GPU networking. UCCL-Tran decouples the data path and control path of existing RDMA NICs and efficiently runs the control-path transport on host CPUs. This software extensibility brings in transport innovations that cannot be achieved in hardware for ML workloads, e.g., a multipath transport to resolve flow collisions. ML collectives atop UCCL-Tran achieve up to 4.5× higher performance compared to existing RDMA NICs.

---

## 论文详细总结（自动生成）

# UCCL-Tran 论文结构化总结

## 1. 核心问题与整体含义（研究动机与背景）

- **根本矛盾**：ML 工作负载（LLM 训练、MoE 服务、PD 分离推理）对网络的需求快速演进，但 RDMA 网卡上的主机网络传输层由硬件固定实现，演进速度远慢于软件，导致应用需求与硬件优化能力之间出现错配。
- **现实痛点实例**：
  - Meta 发现 DCQCN 拥塞控制在低流熵、高突发的 LLM 训练中失效，只能禁用网卡 CC 改在应用层调度；DeepSeek 在 MoE all-to-all 中同样禁用了 CC，但无 CC 的大规模 RDMA 网络存在死锁、队头阻塞与泛洪拥塞风险。
  - 阿里观察到 RDMA 网卡"每连接仅单流/单路径"导致严重流冲突，被迫改用 rail-optimized 双平面拓扑，代价高昂。
  - 此外还有 MoE 服务的网络 incast（最热专家负载可达均值 10 倍）、MLT 等应用-传输协同设计无法落地、go-back-N 重传在丢包下性能差、多厂商网卡异构导致带宽下降 2–33 倍等问题。
- **整体含义**：论文将"网络可扩展性（extensibility）"视为数据中心网络的关键挑战，主张在**不改动现有硬件**的前提下，用纯软件方式为 GPU 网络提供可快速迭代的传输层，使新传输协议研究（多路径、接收端驱动 CC、选择性重传）能够产品化落地。

## 2. 方法论

### 核心思想
将现有 RDMA 网卡的数据路径与控制路径解耦：数据仍由网卡以 GPUDirect 方式高效 DMA 至 GPU 显存，而 CC、可靠性、多路径负载均衡等控制决策全部上移到主机 CPU 的用户态软件中执行。

### 关键技术细节

- **控制/数据路径分离的三种 QP 方案**：
  - **UC（首选）**：绕过硬件 CC 与丢包恢复，同时保留网卡的分段/重组卸载能力；使用 `write_with_imm` 原语，由 32 位 imm_data 承载控制头（8 位连接 ID + 7 位消息 ID + 8 位块序号 CSN + 1 位末块标记 + 8 位预留给接收端驱动 CC）。
  - **RC + 关闭 CC**：用于不支持 UC 的网卡（如 Broadcom Thor-2），但无法自定义硬件可靠性机制。
  - **UD（最后手段）**：用于 AWS EFA（仅有 SRD）。利用 **scatter-gather**（两段 sg_list：CPU 上 64B 固定控制头 + GPU 上数据载荷）让网卡在发送时合并、接收时按固定头长切分，保证控制头与数据"同生共死"；接收端乱序重组则通过**将 scattered memcpy 融合进集合通信原有的 reduction kernel** 完成，避免额外 kernel 启动开销。
  - **非 RDMA 网卡**：基于 AF_XDP 在 UDP 之上构建可靠传输，用不同 UDP 端口实现多路径。

- **多路径机制**：
  - 每对 NIC 默认使用 **256 个 UC/RC QP**，借助 ECMP 以 QP 号为哈希输入获得最多 256 条路径；UD 则用 16×16 收发 QP 组合得到 256 条路径。
  - 同一对 NIC 上的多条连接**共享同一组 QP**，避免 QP 耗尽。
  - 乱序处理：GPU 显存中维护重排序缓冲，采用 seq/ack、更大的重复 ACK 阈值（适应多路径乱序）、快速重传与超时重传；UD 场景由 GPU kernel 完成重排。

- **软件传输效率优化**：
  - **Run-to-completion** 执行模型 + Deficit Round Robin（DRR）调度。
  - **连接拆分（connection splitting）**：将 256 个 QP 均分给多个引擎线程，每个线程维护子连接状态，用 SRQ/SCQ 降低轮询开销，支持单连接跨多核扩展。
  - **控制合并（control coalescing）**：以 32KB 块（而非逐包）做传输决策，兼顾精度与 CPU 效率。
  - **链式 posting（chained posting）**：一次 MMIO 提交最多 32 个 send/recv verb，缓解 UD 的 MMIO 开销。

- **拥塞信号**：受限于网卡不向软件暴露 ECN/trimming 信息，当前实现仅使用**逐路径 RTT（基于网卡硬件时间戳）+ 丢包**作为拥塞信号，并用高优先级专用 ACK QP 降低反馈延迟。

- **三个可扩展性案例**：
  1. **包喷洒多路径传输**：Power-of-Two 采样选最低 RTT 路径 + CUBIC 或 Swift CC（支持逐路径或全局 cwnd）。
  2. **接收端驱动 CC（EQDS）**：每 NIC 一个专用 pacer 线程，维护 rtx/active/idle 三个发送者列表，用 credit UD QP 下发信用；因无 packet trimming，改用超时 + RTS 替代。
  3. **选择性重传**：基于 SACK + bitmap，用 `std::map` 跟踪任意数量乱序包，优于 NVIDIA 新一代网卡的 NACK 小窗口方案。

- **接口设计**：暴露 `onChunkSize`、`onPacingChunk`、`onSelectPath`、`onTxRtxChunk`、`onRxChunk`、`onRxRtxChunk`、`onRxACK`、`onRxCredit` 等回调，供集合通信库/应用开发者扩展。

- **实现规模**：28.4K 行 C++，作为 NCCL/RCCL 网络插件（libnccl-net.so）即插即用；为 EFA 的 UD scattered memcpy 修改约 170 行 NCCL 代码，新增 `irecv_scattered` 与 `irecv_free_ptrs` 接口。

## 3. 实验设计

### 测试平台（4 个）

| 平台 | 服务器数 | 拓扑 | MTU | 网卡 | GPU | CPU |
|---|---|---|---|---|---|---|
| CX_ETH | 6 | 跨机架 fat-tree | 4KB | ConnectX-7 400G×8 | H100-80G×8 | 160 核 AMD |
| AMD | 4 | 跨机架 rail-optimized | 4KB | Broadcom Thor-2 400G×8 | MI300X-192G×8 | 128 核 |
| EFA | 4 | 跨机架 fat-tree | 9KB | AWS EFA 100G×4 | A100-40G×8 | 96 核（p4d.24xlarge） |
| CX_IB | 2 | 同机架 InfiniBand | 4KB | ConnectX-7 400G×8 | H100-80G×8 | 128 核 |

前三个平台租自 TensorWave、IBM Cloud、AWS；通过禁用 NCCL NVLink/SHM 将物理服务器"虚拟化"为更多网络端点以模拟更大集群。

### Benchmark 与对比基线

- **软件栈**：NCCL v2.23.4 + NCCL-tests 9d26b84；AMD 侧用 RCCL 532f54c + RCCL-test 5b27b96。
- **集合通信**：allreduce、all-to-all（主）；allgather、reduce-scatter、multi-collectives（附录）；消息 1MB–1GB。
- **基线**：
  - CX_ETH/CX_IB：NCCL 内置 RC（每连接 4/8/16 QP 扩展）对比 UCCL-Tran UC/RC（256 QP）。
  - AMD：RCCL + Thor-2 原生传输。
  - EFA：官方 AWS OFI NCCL 插件 v1.13.2（SRD 多路径）对比 uTran CUBIC / EQDS / SRD。
- **应用级评估**：
  - DeepSeek-V2-Lite（16B 参数、64 路由专家）在 AMD 平台用 AMD Primus + Megatron-LM 后端做专家并行端到端训练，并扩展到 32/128 专家。
  - DeepSeek-V3 类 MoE 服务：因需数百 GPU，采用含计算时间与隐藏态消息大小的真实 trace 驱动，在 EFA 上用 PyTorch + NCCL 仿真 prefill/decode。
- **专项实验**：15:1 incast 与 16-NIC permutation 共存的 FCT 分布；软件注入 1/256–1/16384 丢包率下的 goodput；htsim 包级仿真（1024×400G NIC、三级 fat-tree、256 路径、64MB 流）评估块大小与 LB 策略；PCIe/MMIO 开销测量（pcm-pcie）。

## 4. 资源与算力

- **GPU 型号与规模**：NVIDIA H100-80G、AMD MI300X-192G、NVIDIA A100-40G，每平台每服务器 8 卡（EFA 为 4×100G 网卡）。
- **集群来源**：均为租用/云上资源（TensorWave、IBM Cloud、AWS p4d.24xlarge），非自建大规模集群；最大实测规模为 6 台服务器，其余通过禁用 NVLink/SHM 模拟。
- **训练时长/总 GPU 小时**：**论文未明确说明** DeepSeek-V2-Lite 训练的具体时长或总算力消耗；应用实验只报告了相对加速比（最高 7.5%）。
- **额外 CPU 开销**：每 NIC 增加 2 个引擎核（每 GPU 2 核），接收端驱动 CC 再加 1 个 pacer 核；DeepSeek-V2-Lite 训练中相对每块 400G 网卡的 CPU 核消耗从 2.3 增至 4.3。

## 5. 实验数量与充分性

### 实验组数概览
- **集合通信主实验**：4 个平台 ×（allreduce + all-to-all）= 8 组主图。
- **附录集合通信**：EFA 上 allgather、reduce-scatter、4 种 multi-collective、NVLink/SHM 开启场景、AF_XDP 非 RDMA 场景，共约 7 组。
- **应用实验**：2 组（训练 + 服务），训练侧含 3 种专家规模。
- **可扩展性案例**：incast FCT、丢包 goodput、CPU 核数（CX_IB 与 EFA 各一组）、GPU 数量、路径数量、块大小，约 8 组。
- **消融/机理分析**：QP 数量扫描（60–60K）、连接拆分、CC 决策延迟与 ACK 周转延迟、htsim 块大小×LB 策略矩阵、kernel fusion vs kernel launching、PCIe/MMIO 开销，约 6 组。
- **合计约 30 组以上实验**，覆盖多厂商网卡、多 GPU 架构、多集合通信类型与多应用场景。

### 充分性与客观性评估
- **优点**：跨 NVIDIA/Broadcom/AWS 三类网卡与 H100/MI300X/A100 三类 GPU 验证，泛化性论证较强；同时给出应用级端到端收益，而非仅微基准；对软件传输的固有代价（CPU 核、PCIe 开销、CC 延迟）做了诚实量化。
- **需注意的偏差风险**：
  - 大量"大规模"结果是靠**禁用 NVLink/SHM 模拟**得到的
