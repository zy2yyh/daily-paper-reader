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
evidence: 解耦RDMA数据与控制路径的软件传输层，多路径传输优化集合通信
tldr: 现有RDMA网卡的主机网络传输难以演进，单路径RDMA流量易发生流冲突，严重降低机器学习集合通信性能。UCCL-Tran提出可扩展的软件传输层，将RDMA网卡的数据路径与控制路径解耦，把控制路径传输高效运行在主机CPU上，从而实现多路径传输等硬件难以支持的传输创新。实验表明基于UCCL-Tran的机器学习集合通信吞吐最高提升4.5倍，为GPU网络提供了可持续演进的通信基础。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: RDMA网卡上的主机传输难以演进，单路径流量易发生流冲突，严重制约机器学习集合通信性能。
method: 将RDMA网卡的数据路径与控制路径解耦，在主机CPU上高效运行控制路径传输并引入多路径机制。
result: 基于该传输层的机器学习集合通信吞吐最高提升4.5倍，显著缓解单路径流冲突问题。
conclusion: 该工作表明软件可扩展传输层能为GPU网络提供可持续演进的低延迟通信基础。
---

## Abstract
Fast-evolving machine learning (ML) workloads have increasing requirements for networking. However, host network transport on RDMA NICs is hard to evolve, causing problems for ML workloads. For example, single-path RDMA traffic is prone to flow collisions that severely degrade collective communication performance. We present UCCL-Tran, an extensible software transport layer to evolve GPU networking. UCCL-Tran decouples the data path and control path of existing RDMA NICs and efficiently runs the control-path transport on host CPUs. This software extensibility brings in transport innovations that cannot be achieved in hardware for ML workloads, e.g., a multipath transport to resolve flow collisions. ML collectives atop UCCL-Tran achieve up to 4.5× higher performance compared to existing RDMA NICs.

---

## 论文详细总结（自动生成）

# UCCL-Tran 论文结构化总结

## 一、核心问题与整体含义

- **背景张力**：ML 负载的网络需求演进极快——从参数服务器/百卡 allreduce，到千卡 LLM 训练的多级并行与多种集合通信，再到 prefill-decode 分离的点对点通信，直至 MoE 服务中数百卡间的 all-to-all。但 **RDMA 网卡上的主机传输层是硬件固化的**，硬件迭代周期远长于软件，导致应用需求与硬件优化错配。
- **现实痛点（论文给出的六个例子）**：
  - Meta 发现 DCQCN 对低熵、高突发的 LLM 训练流量效果差，被迫在网卡上**关闭 CC**、改由应用层调度；DeepSeek 在 MoE all-to-all 中也关闭 CC。但无 CC 的大规模 RDMA 网络脆弱，易引发死锁、队头阻塞与普遍拥塞。
  - 阿里观察到单路径 RDMA 造成严重**流冲突**，不得不重建为 rail-optimized 双平面拓扑，代价高昂——而纯软件多路径本可避免。
  - MoE 服务中专家热点（最热专家负载可达均值 10×）造成**瞬时 incast**，EPLB 的重平衡周期约 10 分钟，来不及响应，需要接收端驱动 CC，而商用网卡不提供。
  - **应用-传输协同设计**（如 MLT 的半可靠梯度传输）因网卡可编程性不足无法落地。
  - **低效丢包恢复**：网卡受片上 SRAM 限制只能 go-back-N，被迫依赖 PFC，而 PFC 会带来死锁与牺牲流。
  - **异构网卡**：跨代/跨厂商通信带宽可下降 2–33×。
- **核心问题**：能否在**不修改现有商用 RDMA 网卡硬件**的前提下，构建一个可扩展（可快速演进传输协议）且性能达到硬件级的 GPU 网络传输层？
- **整体含义**：把"控制路径"从硬件搬到主机 CPU 用户态，使新传输协议（多路径、接收端驱动 CC、选择性重传）能够快速产品化；作者希望以此"打开 ML 传输研究落地之门"。

## 二、方法论

### 1. 核心思想：数据路径与控制路径解耦

- **数据路径**：仍由网卡 DMA 直接写入 GPU 显存（GPUDirect），保持零拷贝与高吞吐。
- **控制路径**：CC、负载均衡、丢包恢复、乱序处理等决策全部在主机 CPU 用户态完成，因此可用普通 C++ 代码修改、重编译、插桩。
- **部署形态**：作为 NCCL/RCCL 的 net plugin（`libnccl-net.so`）编译加载，应用无需改代码或重编译（UD 路径例外，需约 170 LoC 改动）。

### 2. 分离控制与数据的具体机制（按网卡能力分级）

| 优先级 | QP 类型 | 机制 | 适用 |
|---|---|---|---|
| 首选 | **UC** | `write_with_imm` 传递 32-bit `imm_data` 作控制头；网卡负责分段/重组，但绕过硬件 CC、可靠性与乱序处理 | NVIDIA ConnectX 等 |
| 次选 | **RC + 关闭 CC** | 同样用 `write_with_imm`；可获更快 ACK 与更精确 RTT，但丢包恢复仍被硬件固化 | Broadcom Thor-2 |
| 兜底 | **UD / SRD** | `send/recv` 两段 `sg_list`（CPU 侧 64B 控制头 + GPU 侧 payload），由网卡自动合并/拆分，保证控制头与数据"同生共死" | AWS EFA |
| 非 RDMA | **AF_XDP + UDP** | 用户态自建可靠传输，CPU 重组后 `cudaMemcpy` 到 GPU | AWS ENA |

- **32-bit 控制头字段划分**：8 bit 连接 ID + 7 bit 消息 ID（每 NIC 对 256 连接、每连接 128 在途消息）+ 8 bit chunk 序号（CSN）+ 1 bit 末 chunk 标记 + 8 bit 预留给接收端驱动 CC。
- **UD 的两个独有难题及解法**：
  - *控制头分离*：用 scatter-gather 让网卡在发送时合并、接收时按固定头长切分，避免 CPU 去 GPU 显存读头。
  - *乱序重组*：UD 无重组卸载，作者把 scattered memcpy **融合进集合库已有的 reduction kernel**（先散拷入 transport buffer，再做归约），避免额外 kernel launch；代价仅为额外显存带宽，受网络带宽约束，可忽略。

### 3. 多路径与乱序

- **多路径来源**：RoCE/IB 的 ECMP 以 QP 号为哈希输入，故每连接默认用 **256 个 UC/RC QP**（最多 256 条路径）；UD 用 16 源 ×16 目的 = 256 组合；AF_XDP 用不同 UDP 端口。
- **QP 复用**：同一对 NIC 间的所有连接（含多 GPU 共享一网卡）**共享同一组 QP**，避免 QP 耗尽。
- **乱序处理**：seq/ack + 快重传（阈值大于 TCP 的 3，以适应多路径乱序）+ 超时重传；**重排序缓冲放在 GPU 显存**，由网卡直接 DMA 写入。

### 4. 高效软件传输的四项技术

- **Run-to-completion + DRR 调度**：单 engine 线程内串行完成 RX/TX/ pacing/超时/重传，用 Deficit Round Robin 在多功能与多连接间公平复用。
- **Connection splitting**：把 256 个 QP 均分给多个 engine 线程，形成子连接，各有独立 CC/LB 状态；用 SRQ/SCQ 降低轮询开销；应用侧按"最不忙 engine"派发消息。
- **Control coalescing**：以 **32KB chunk** 为控制决策粒度（而非逐包），理由是 CC 本就按 per-RTT 而非 per-ACK 反应；拥塞加剧时可自适应缩小 chunk。
- **Chained posting**：UD 下一次 MMIO 投递最多 32 个 WQE，缓解无卸载带来的 MMIO 开销。

### 5. 拥塞信号与保真度

- 网卡消费掉含 ECN/trimming 的包头，软件只能拿到 payload，故仅用 **per-path RTT（硬件时间戳）+ 丢包**作为信号。
- 降低决策延迟的手段：ACK 走专用高优先级 QP（DSCP）、优先轮询其 CQ、DRR 中给 ACK 更高预算；NIC-CPU 时钟同步间隔 100 μs（仿 Swift）。

### 6. 可扩展接口与三个案例

- **接口**：8 个回调 `onChunkSize / onPacingChunk / onSelectPath / onTxRtxChunk / onRxChunk / onRxRtxChunk / onRxACK / onRxCredit`。
- **案例一（包喷洒多路径）**：Power-of-Two 采样选最低 RTT 路径，CC 用 CUBIC 或 Swift，支持 per-path 或全局 CC 状态。
- **案例二（接收端驱动 EQDS）**：每 NIC 一个专用 pacer 线程，维护 rtx / active / idle 三张发送者链表，用 credit UD QP 下发信用；因无 packet trimming，用 timeout + RTS 替代。
- **案例三（选择性重
