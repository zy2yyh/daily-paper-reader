---
title: "{RoCE} {BALBOA}: {Service-Enhanced} {RDMA} Offload Engine for Data Center {SmartNICs}"
title_zh: RoCE BALBOA：面向数据中心SmartNIC的服务增强RDMA卸载引擎
authors: "Maximilian Jakob Heer, Benjamin Ramhorst, Yu Zhu, Luhao Liu, Zhiyi Hu, Jonas Dann, Gustavo Alonso"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-heer.pdf"
tags: ["query:dcn-comm"]
score: 9.0
evidence: 面向数据中心SmartNIC的开源100Gbps RDMA卸载引擎，可修改传输层
tldr: 论文针对RDMA虽已成为数据中心高性能网络的事实标准、但部署依赖固定功能的商用黑盒网卡、研究者无法修改传输层以适配专用任务的问题，提出开源100Gbps RDMA卸载引擎BALBOA，兼容商用RNIC并支持自定义传输层与完整的卸载网络栈。该引擎让新的网络机制能够在接近真实的硬件环境中实现与验证。其意义在于为数据中心RDMA传输层研究提供可编程、可复现的硬件平台。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-001.webp\", \"caption\": \"\", \"page\": 6, \"index\": 1, \"width\": 512, \"height\": 512}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-002.webp\", \"caption\": \"\", \"page\": 7, \"index\": 2, \"width\": 8779, \"height\": 8795}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-003.webp\", \"caption\": \"\", \"page\": 8, \"index\": 3, \"width\": 786, \"height\": 333}]"
motivation: RDMA已成数据中心高性能网络事实标准，但商用网卡是固定功能黑盒，研究者无法修改传输层以适配专用任务。
method: 提出开源100Gbps RDMA卸载引擎BALBOA，兼容商用RNIC，可自定义传输层并提供完整的卸载网络栈。
result: 填补研究用网卡缺乏真实卸载网络栈的空白，使新网络机制可在接近生产的环境中验证。
conclusion: 为数据中心RDMA传输层的研究与定制提供了可编程、可复现的硬件平台。
---

## Abstract
Remote Direct Memory Access (RDMA) has become the de facto standard for high-performance data center networking. However, current deployments rely heavily on fixed-function, commercial NICs. These "black box" commercial hardware implementations prevent researchers and system architects from modifying the transport layer for specialized tasks. In parallel, research on NICs often lacks offloaded networking stacks or uses simplified protocol implementations, limiting insight into novel networking solutions in realistic settings. In this paper, we bridge this gap by introducing BALBOA, an open-source, 100 Gbps RDMA offload engine designed for research on networking and fully compatible with commercial RNICs. Unlike prior stack implementations which lack scalability and bandwidth, or struggle with data center interoperability and miss strict protocol compliance, BALBOA supports hundreds of Queue Pairs in switched network environments and allows for line-rate offloads, making it a viable platform for realistic data center research. We describe the system architecture, detailing how BALBOA overcomes FPGA memory and timing bottlenecks through a decoupled state architecture and streaming control-data separation. We evaluate BALBOA on a hardware cluster with FPGAs, RNICs, and switches, showing that it matches the performance of commercial ASICs while offering full customization. Finally, we showcase BALBOA’s potential through novel case studies: protocol enhancements for infrastructure purposes (encryption, deep packet inspection) and an offloaded preprocessing pipeline for deep learning recommender systems, which applies streaming transformations to the incoming data before feeding it directly to a GPU for model serving.

---

## 论文详细总结（自动生成）

# RoCE BALBOA：面向数据中心 SmartNIC 的服务增强 RDMA 卸载引擎（OSDI 2026，ETH Zurich）

## 1. 核心问题与研究动机

- **背景**：RDMA 已成为数据中心高性能网络的事实标准（据文中引用，占云网络流量可达 70%），通过将网络栈卸载到网卡、绕过主机 OS，实现低延迟、高吞吐与低 CPU 占用，支撑内存解耦、分布式 KV 存储、数据分析等系统。
- **核心矛盾一：商用 RNIC 是"黑盒"**。传输层逻辑硬连线，无法加入流跟踪状态机、遥测、加密、访问控制等；RDMA 本身缺少访问控制与加密策略（ReDMArk 等工作已揭示安全缺陷）；学术界提出的大量拥塞控制方案在商用卡上通常只有 1–2 种固化实现。要修改只能落到软件（丧失 RDMA 性能优势）或离路协处理器（引入显著延迟）。
- **核心矛盾二：SmartNIC/DPU 仍不够"开放"**。AWS Nitro、NVIDIA BlueField、Meta FBNIC 用卡载 ARM 核做卸载，属"look-aside"模型，需把包搬进核的内存层级，100 Gbps 下复杂应用难以线速；传输层仍是硬件黑盒。
- **核心矛盾三：现有 FPGA 方案存在明显短板**。学术/商用 FPGA 平台（Corundum、PANIC、OpenNIC、StRoM、Schelten 等）或缺少卸载网络栈（依赖主机 CPU）、或吞吐不足 100G、或缺少 CRC/重传等复杂部件导致不兼容商用 RNIC、无法过交换机、无 GPU-Direct DMA。
- **论文含义**：BALBOA 试图证明此前被视为"根本对立"的三件事——**100G 线速、严格 RoCEv2 协议合规、数据通路可扩展性**——可以在架构层面同时成立，从而为数据中心 RDMA 传输层研究提供开源、可复现、可插拔的硬件平台（可作为 SmartNIC/DPU 的即插即用组件）。

## 2. 方法论

### 2.1 设计目标与取舍
- 四条需求：**R1** 100G 吞吐（可升级 200G）；**R2** 协议合规（RoCEv2、CRC、健壮重传，可与商用卡/交换机互通）；**R3** 资源/功耗高效（为卸载逻辑留空间）；**R4** 开放可扩展（模块化 + 标准接口）。
- 功能子集取舍：仅实现 **RC 连接模式 + 单边 RDMA WRITE / READ**（依据既有研究，这是最常用且硬件最复杂的配置，含重传、超时、有序交付）；UC 模式被论证为"门控可靠性逻辑 + 扩展 opcode 表"的简单延伸。

### 2.2 整体架构
- 由五类模块组成：**报文处理流水线（IP/UDP/IB 头）**、**ICRC 校验**、**重传缓冲**、**流控**、**仲裁**；模块间统一使用工业标准 **AXI4-Stream**（数据流 + 控制流）互联，便于单独仿真验证与深度流水化。
- 数据通路 **512 bit @ 250 MHz = 128 Gbps**，满足 100G 并留余量；整套设计可综合至 **400 MHz**，为 200G 预留升级路径。
- 用 **Vitis HLS** 实现，按"纵向（逐协议头）× 横向（RX/TX）"划分流水级；相比 RTL 显著降低开发/移植门槛，并借 HLS 自动流水满足时序与资源约束。
- 已移植到 Coyote v2 shell（XDMA + 100G CMAC），并有向 AMD RecoNIC/ONIC 移植的工作，验证可移植性。

### 2.3 关键设计一：解耦状态架构
- 连接表、PSN 表、MSN 表放在**双端口 BRAM**，QP 建立时初始化，使 RX 与 TX 流水线可**同时无阻塞访问**，避免双向 100G 下的状态一致性停顿。
- 默认支持 **500 QP**（可配置）；共享传输定时器负责超时触发重传，并可为未来基于 RTT 的拥塞信号提供输入。

### 2.4 关键设计二：控制流/数据流分离 + 三态接口
- 接口分 **data / control / completion** 三路：数据在数据通路上直通，控制命令在并行控制总线上操作与同步，避免"扩容型数据操作"下统一传输的复杂性。
- **RX 路径**：逐层剥离头部，由专用 FSM 生成轻量 meta-command 做状态读取与检查；BTH/RETH 阶段在专用 completion 总线上产生完成事件，使流控成为**可独立替换的模块**。
- **TX 路径**：从主机接收独立的数据流与命令流，命令流取 QPN/PSN 生成 RETH/BTH 头，再与 payload 逐级交织，最终送入 ICRC 与 CMAC；解耦使仲裁延迟、重传取数不会阻塞头部生成。

### 2.5 关键设计三：HBM 支撑的重传与流复用
- 本地 WRITE 与远端 READ 的数据投递特性相反，混在单流会造成停顿；BALBOA 将 **WRITE 数据与 READ 响应放在不同 AXI 流**，再经仲裁汇入单流，从而同时达到双向 100G，并允许为 READ/WRITE 部署不同卸载逻辑。
- 所有已发送 payload 在收到 ACK 前缓存在**专用直连 HBM 通道**（片上 SRAM 无法容纳 100G 的带宽时延积，从主机重取又太慢）。
- 流控允许每流 **N = 16 个在途包**，4K MTU 下每流重传缓冲约 **64 kB**；U55C 的 32 GB HBM 理论上可支撑约 **50 万 QP**，反而优于商用 RNIC 的几 MB 片上 SRAM。
- 实测从 HBM 取回一个 MTU payload 需 **1.732 μs**，从触发重传到完整包发出的总延迟 **1.86 μs**，仅为该尺寸包 RTT 的一小部分。

### 2.6 关键设计四：流控与 ICRC
- **流控**：基于 ACK 的固定大小滑动窗口，按 PSN 标识；每 QP 的剩余配额存于 BRAM，发包递减、收 ACK 递增；支持超时与 PSN 触发的 **Go-Back-N** 重传，兼容商用卡。流控作为独立块，可用 DCQCN、TIMELY 等替换，并可通过 FPGA 部分重构实现运行时替换；底层 CMAC 原生支持 PFC。
- **ICRC**：性能关键且组合逻辑复杂，采用"逐 AXI beat 计算部分校验并拼接"的思路，为 **512-bit 全拍**、**320-bit 部分拍**（RoCE 在 512-bit 总线上的常见填充）各建并行流水，再加一条 **32-bit 多级流水**覆盖任意 beat（依赖 RoCE 消息 4 字节对齐保证）。

### 2.7 卸载模型（三类插槽 + 应用卸载）
- **A 型：数据通路上的协议增强**（位于流水线前/后，纯流式，不干预协议逻辑），例：AES-CTR。
- **B 型：路径内增强**（嵌入流水线内部，如头部混淆）。
- **C 型：并行路径增强**（对报文副本运算，通过 HLS hook 与流水线交互），例：ML-DPI；其延迟被流水线部分延迟掩盖，是开放栈独有的能力。
- **D
