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
evidence: 面向低时延数据中心传输的开源100Gbps RDMA卸载引擎
tldr: RDMA是数据中心高性能网络的事实标准，但商用网卡功能固定且传输层封闭，研究者难以定制协议或验证新机制。BALBOA提出开源100Gbps RDMA卸载引擎，提供可修改的传输层与完整卸载网络栈，并兼容商用RNIC。该平台使研究者能在真实硬件上实验新型RDMA传输设计。它为低时延数据中心通信的传输层创新提供了开放基础。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-001.webp\", \"caption\": \"\", \"page\": 6, \"index\": 1, \"width\": 512, \"height\": 512}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-002.webp\", \"caption\": \"\", \"page\": 7, \"index\": 2, \"width\": 8779, \"height\": 8795}, {\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-roce-balboa-service-enhanced-rdma-offload-engine-for-data-center-smartnics/fig-003.webp\", \"caption\": \"\", \"page\": 8, \"index\": 3, \"width\": 786, \"height\": 333}]"
motivation: RDMA已成为数据中心高性能网络的事实标准，但商用网卡固件封闭，无法修改传输层以支持专门任务，研究也缺乏真实协议栈。
method: 实现开源100Gbps RDMA卸载引擎，提供可修改的传输层与完整卸载网络栈，并与商用RNIC保持兼容。
result: 在真实硬件上支持对RDMA传输层的定制与实验，达到100Gbps并兼容现有商用部署。
conclusion: 为数据中心RDMA传输的研究与创新提供了开放、可扩展的硬件平台。
---

## Abstract
Remote Direct Memory Access (RDMA) has become the de facto standard for high-performance data center networking. However, current deployments rely heavily on fixed-function, commercial NICs. These "black box" commercial hardware implementations prevent researchers and system architects from modifying the transport layer for specialized tasks. In parallel, research on NICs often lacks offloaded networking stacks or uses simplified protocol implementations, limiting insight into novel networking solutions in realistic settings. In this paper, we bridge this gap by introducing BALBOA, an open-source, 100 Gbps RDMA offload engine designed for research on networking and fully compatible with commercial RNICs. Unlike prior stack implementations which lack scalability and bandwidth, or struggle with data center interoperability and miss strict protocol compliance, BALBOA supports hundreds of Queue Pairs in switched network environments and allows for line-rate offloads, making it a viable platform for realistic data center research. We describe the system architecture, detailing how BALBOA overcomes FPGA memory and timing bottlenecks through a decoupled state architecture and streaming control-data separation. We evaluate BALBOA on a hardware cluster with FPGAs, RNICs, and switches, showing that it matches the performance of commercial ASICs while offering full customization. Finally, we showcase BALBOA’s potential through novel case studies: protocol enhancements for infrastructure purposes (encryption, deep packet inspection) and an offloaded preprocessing pipeline for deep learning recommender systems, which applies streaming transformations to the incoming data before feeding it directly to a GPU for model serving.

---

## 论文详细总结（自动生成）

# RoCE BALBOA 论文结构化总结

## 1. 核心问题与研究动机

- **背景**：RDMA 已成为数据中心高性能网络的事实标准（云网络流量占比可达 70%），通过将网络栈卸载到网卡、绕过主机 OS 实现低延迟、高吞吐与低 CPU 占用，是内存解耦、分布式 KV 存储、大数据分析等系统的基石。
- **核心矛盾一：商用 RNIC 是"黑盒"**。传输层逻辑硬连线，无法添加流跟踪状态机、遥测、加密等；RDMA 本身沿用可信 HPC 域假设，缺乏访问控制与密码学策略，而已有的多种拥塞控制研究方案在商用网卡上通常只有一两种固化实现。任何修改只能退化为软件实现（丧失性能优势）或放到慢速旁路协处理器（引入显著延迟）。
- **核心矛盾二：现有可编程平台各有短板**。SoC 型 SmartNIC/DPU（AWS Nitro、BlueField、Meta FBNIC）虽可用 ARM 核做应用卸载，但存在 "look-aside" 瓶颈（需将包搬入核的内存层次），100Gbps 线速下复杂应用不可行，且传输层仍是硬件黑盒；学术/商用 FPGA 平台（Corundum、PANIC、OpenNIC、StRoM 等）往往缺卸载网络栈、吞吐不足 100G、缺少 CRC/重传等复杂部件导致不兼容 RoCEv2 规范、无法在交换机网络中通信、缺 GPU-direct RDMA。
- **论文含义**：BALBOA 试图证明此前被认为互相冲突的三个属性——**100G 线速、严格协议合规、数据通路可扩展性**——可以在架构层面同时成立，从而为数据中心网络研究提供"可插拔"的真实硬件基础设施。

## 2. 方法论

### 2.1 设计需求（R1–R4）
- **R1** 100G 吞吐（并为 200G 留升级路径）；**R2** RoCEv2 协议遵循与商用网卡/交换机互操作（需健壮快速重传与计算复杂的 ICRC）；**R3** 资源/功耗开销最小，为卸载留面积；**R4** 开放可扩展，模块化并暴露标准接口。
- 功能子集选择：仅实现 **RC 模式 + 单边动词（RDMA WRITE / READ）**，理由是这是数据中心最常用配置且硬件支持最复杂（含重传、超时、有序交付）；UC 模式可通过门控可靠性逻辑与扩展 opcode 表平滑扩展。

### 2.2 架构要点
- **模块化流水线**：按协议头垂直划分（IP/UDP/IB）、按 RX/TX 水平划分，用 AMD Vitis HLS 实现；组件间统一走 **AXI4-Stream**（数据与控制分离），便于独立仿真、深度流水与 timing closure。
- **位宽/频率**：512-bit @ 250 MHz = 128 Gbps 余量满足 100G；完整设计可综合至 400 MHz，为 200G 留路径。
- **解耦状态架构**：连接表、PSN 表、MSN 表放在**双端口 BRAM**，QP 建立时初始化，RX/TX 流水线可同时访问而不互相阻塞；默认支持 **500 QP**，可配置扩展；共享传输定时器负责超时重传并可生成 RTT 拥塞信号。
- **三流接口**：data / control / completion 三套流，避免在统一流上做"尺寸扩张型"命令操纵；BTH/RETH 阶段在专用总线上产生完成事件，使流控成为可独立替换的模块。
- **重传与流复用**：将 **RDMA WRITE 数据与 READ RESPONSE 数据分置两条 AXI 流**，避免单流互斥导致停顿，并允许对 READ/WRITE 部署不同卸载；所有已发载荷缓存于**直连 HBM 通道**（而非片上 SRAM，因 100G BDP 过大；也不回读主机，因 PCIe 往返延迟不可接受）。按 Go-Back-N 语义，每流最多 N=16 个未确认包，4K MTU 下约 64 kB/QP；32 GB HBM 理论可支撑约 50 万 QP。实测从 HBM 取回一个 MTU 载荷需 1.732 μs，从触发重传到完整包发出的总延迟 1.86 μs。
- **流控**：基于 ACK 的固定窗口滑动（按 PSN 标识），独立于包处理流水线的控制通路，剩余包预算存于 BRAM，发包递减、收 ACK 递增；该模块可整体替换为 DCQCN/TIMELY 等算法，未来可结合 FPGA 部分重构实现运行时可编程拥塞控制；底层 CMAC 原生支持 PFC。
- **ICRC 流水线**：按 AXI beat 逐拍累加部分校验和再拼接；针对 512-bit 全拍、320-bit 部分拍、32-bit 分块分别建并行流水（利用 RoCE 消息 4 字节对齐保证），避免片外查表延迟。
- **调试设施**：100G 双向、性能无损的流量嗅探器，输出标准 PCAP，支持 CMAC 层按协议头过滤与去载荷。
- **卸载方法论（三档）**：
  - *On-datapath 协议增强*（如 **AES-CTR**，置于流水线前后，11 周期/44 ns 固定延迟、线速；关键优势是栈向加密逻辑暴露传输状态，可用 **QPN + 每包计数器**构造唯一 IV，无需软件介入；也可移位到流水线内部加密 BTH/UDP 头）。
  - *Parallel-path 协议增强*（如 **ML-DPI**，与主流水线并行处理复用副本，通过 HLS hook 与 BTH 处理函数交互，把判决作为标志写入主机命令或直接丢包；推理延迟 44 ns/beat，短于流水线延迟故被完全隐藏）。
  - *应用级卸载*（用户任意逻辑挂在 RX/TX 数据通路，可使用数据与控制流做 scatter-gather、转发到板载内存、用 MMIO 寄存器提供状态；约束是 RX 侧不得停顿、扩张型操作受背压限制；提供与真实网络行为一致的仿真框架，部署与仿真仅差一个编译开关）。
- **可移植性**：评估基于开源 Coyote v2 shell + XDMA + 100G CMAC，已有移植到 AMD RecoNIC/ONIC 的工作。

## 3. 实验设计

- **平台/场景**：ETH 公开研究集群 HACC 的 100G 子网，AMD Alveo U55C FPGA 作为 PCIe 卡，对端为 Mellanox ConnectX-5（MCX515A-CCAT）与 NVIDIA ConnectX-7（BlueField-3 B3220-DPU），经 Cisco Nexus 9000 交换机，4K MTU；除单层交换外还在 **2 层 fat-tree** 拓扑复核；另在 U250/U280 上成功部署。DLRM 用例使用 AMD MI210 GPU 与 EPYC 7V13；加密基线用 EPYC 7302P（16 核）+ ConnectX-5。
- **基准与对比组**：
  - 基础性能：BALBOA↔BALBOA、CX5↔CX5、CX7↔CX7、CX5↔BALBOA、CX7↔BALBOA 五组曲线，覆盖 256 B–128 KB 缓冲，吞吐与延迟，每测点 100 次重复并给出 95 分位。
  - 多 QP：1/2/4/8/16/32 QP 交错 READ 的带宽分配与聚合。
  - 协议增强：AES-CTR 对比主机 CPU OpenSSL 实现（含 16 核）；DPI 开/关对比；DPI 精度引用配套工作。
  - 资源：post-routing 资源与功耗估计，并与开源 100G TCP 栈 Limago 在等效条件（500 流、U55C）下对比。
  - 应用用例：三种配置——① CPU 软件预处理（1/2/4/8 线程）+ 拷入 GPU；② FPGA 预处理 + 经主机内存拷贝；③ FPGA 预处理 + 直连 GPU DMA。
- **未完成的对比**：尝试与 BlueField-3 的硬件 IPsec 对比，因官方路径依赖打过补丁的 strongSwan（文档稀疏、依赖与时间戳问题）而失败，NVIDIA 论坛亦无有效答复，故缺少与商用 DPU 加密路径的直接数据。

## 4. 资源与算力

- **未涉及模型训练算力**：本文不训练模型；DPI 模型为已训练并量化的全连接三值网络（细节引 [44]），论文未给出训练所用 GPU 型号、数量或时长。这一信息缺口需明确指出。
- **FPGA 资源（U55C，post-routing 估计）**：BALBOA 基础栈 43,732 LUT（3.4%）、101 BRAM（5.1%）、102,988 FF（4%）、**1.745 W**；其中 ICRC 12,462 LUT、包处理 29,897 LUT、重传逻辑 587 LUT、流控 787 LUT；状态表均驻留片上（Conn/State/Timer/MSN 合计 <600 LUT、17 BRAM）。
- **卸载开销**：AES 65,662 LUT（5%）、0.356 W；ML-DPI 54,404 LUT（3.75%）、3.231 W；两者同时启用后总 LUT 仍 **<12.15%**。
- **对比**：相比 Limago 100G TCP 栈，LUT 少 18%，但 FF（102,988 vs 72,974）与 BRAM（101 vs ~78 BRAM36 当量）更多，仍远在预算内。
- **平台约束**：GPU–FPGA 路径上的 PCIe switch 将带宽上限压到 70 Gbps（8500 MB/s），这直接限定了第 8 节用例的吞吐天花板。

## 5. 实验数量与充分性

- **规模**：约 6 组主要实验（基础吞吐/延迟 × 读写、多 QP 扩展、AES、DPI、资源与功耗、DLRM 预处理），加上 2 层拓扑复核与多板卡部署验证，整体覆盖"栈性能—协议增强—应用卸载—资源开销"四个层面，在同类系统论文中属较为完整。
- **客观性**：全部在公开集群上以商用网卡同台对照，包含同构与异构（FPGA↔ASIC）连接，采用 100 次重复 + 95 分位以体现尾延迟，资源数据为 post-routing 而非综合估算，这些做法较为严谨；对 READ 中等尺寸落后于 ASIC 的原因给出了机制性解释（250 MHz 逻辑频率

限制重传与流水线节拍、以及 READ 需往返取数导致小包开销摊薄不足），而非笼统归因于"硬件不如 ASIC"。

- **可质疑之处**：
  - **功能子集偏窄**：仅 RC + 单边动词，未覆盖 SEND/RECV、原子操作、UC/UD、内存窗口与保护域等；对"协议合规"的论证因此是"子集内合规"，虽与商用网卡互通成功，但不能外推为完整 RoCEv2 兼容。
  - **规模实验有限**：默认 500 QP，多 QP 实验最多到 32 QP 且为单机对单机，缺少数百 QP 并发、多打一 incast、真实数据中心多跳拥塞场景下的表现；交换机侧 PFC/ECN 交互未做系统评测。
  - **基线选择不均衡**：与商用网卡对比充分，但与同类可编程 RDMA 原型（如 Corundum 上的 RoCE 实现、StRoM、PANIC 等）缺少同台定量对比，只有与 Limago（TCP 栈）的资源对比；因此"相对学术 SOTA 的改进幅度"不完全可判定。
  - **缺失的对比被明确承认**：BlueField-3 硬件 IPsec 对比失败，使"可编程加密 vs 商用 DPU 加密"的核心卖点缺少直接数据支撑，只与主机 CPU OpenSSL 比较，而后者的能效与延迟本就不是同一设计目标。
  - **应用用例受平台瓶颈压制**：DLRM 用例受 70 Gbps PCIe 上限约束，加速比无法反映 100G 栈的真实潜力，削弱了"卸载带来端到端收益"的说服力。
  - **单点测量与统计**：100 次重复 + 95 分位属合理但偏轻量，未报告方差、置信区间或跨多次运行的稳定性；功耗为估计值而非实测墙插功率。

## 6. 主要结论与贡献

- **核心论断得到验证**：100G 线速、RoCEv2 合规互通、数据通路可扩展三者在同一 FPGA 架构上可同时成立。基础栈仅占 U55C 约 3.4% LUT、1.745 W，为卸载留下绝大部分面积与功耗预算。
- **性能定位**：与 ConnectX-5/7 同台时，大包吞吐接近线速、小包与中等尺寸存在可解释差距（逻辑频率 250 MHz、重传缓冲与流水线节拍约束），整体处于"可用而非超越 ASIC"的区间——这与论文自身定位一致：目标不是替代商用网卡，而是提供可修改的真实硬件基座。
- **协议增强的可行性证据**：AES-CTR 以 11 周期固定延迟、线速运行，且因栈向加密逻辑暴露 QPN 与包计数器而免去软件协商 IV，这是纯旁路方案难以获得的优势；ML-DPI 以 44 ns/beat 推理延迟被流水线延迟完全隐藏，证明"并行路径 + HLS hook"这一卸载范式对低延迟推理有效。
- **卸载方法论的贡献**：提出 on-datapath / parallel-path / 应用级三档分类，并给出各自约束（RX 不得停顿、扩张型操作受背压限制），把"如何在可编程网卡上放置功能"从个案经验上升为可复用的设计框架，这一部分是论文最具迁移价值的内容。
- **工程贡献**：解耦状态架构（BRAM 双端口连接/PSN/MSN 表）、READ 与 WRITE 分流的双 AXI 流以避免流互斥停顿、基于 HBM 的重传缓冲（1.86 μs 重传恢复延迟）、并行 ICRC 流水、以及性能无损的 100G PCAP 嗅探器，均是可直接被后续工作复用的构件。

## 7. 局限性与未来方向

- **协议覆盖面**：需补齐 SEND/RECV、原子动词、UC/UD、保护域与内存窗口，才能称完整 RoCEv2 兼容；论文给出的"门控可靠性逻辑 + 扩展 opcode 表"路径尚属设想，未实现验证。
- **规模与拥塞**：500 QP 默认容量与 32 QP 实测远低于现代数据中心单机数千连接的现实；基于 ACK 的固定窗口流控在大规模 incast 下是否稳定、与 ECN/PFC 的交互如何，均待评估。论文提出可替换为 DCQCN/TIMELY 并结合部分重构实现运行时可编程拥塞控制，这是明确但未完成的议程。
- **PCIe 与主机路径**：GPU-direct 路径受 PCIe switch 限制在 70 Gbps，未来需 PCIe Gen5/更大 lane 配置或 CXL 类互连才能释放栈的 100G 能力。
- **工具链与可移植性**：依赖 Vitis HLS 与 Coyote shell，已移植到 RecoNIC/ONIC 但仅验证可用性；HLS 对时序收敛与资源可预测性的影响、以及非专家用户编写卸载模块的门槛，论文未做用户研究。
- **安全与多租户**：加密虽有实现，但密钥管理、每 QP 策略下发、以及与可信执行环境的协同未展开；单 FPGA 多租户隔离与故障域问题也未涉及。

## 8. 总体评价

这是一篇定位清晰、工程扎实的系统论文：它不追求在绝对性能上击败商用 RNIC，而是针对"RDMA 传输层不可编程"这一真实且长期存在的痛点，交付了一个线速、合规、模块化且开销极低的开源硬件基座，并用三档卸载范式与两个真实增强（线速加密、隐藏延迟的 DPI）证明其可用性。实验在公开集群上与 ConnectX-5/7 同台对照、采用 post-routing 资源数据与尾延迟统计，方法学较为规范；对落后项给出了机制性解释而非回避。主要不足在于协议子集偏窄、规模实验保守、缺少与同类可编程 RDMA 原型的定量对比，以及关键商用 DPU 加密对比的缺失，使"可编程性收益"的量化论证留有余地。总体而言，其价值更多在于**为后续数据中心网络研究提供可修改的真实基础设施与方法论**，而非某一项指标的领先。

（完）
