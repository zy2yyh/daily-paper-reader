---
title: High-throughput and Flexible Host Networking for Accelerated Computing
title_zh: 面向加速计算的高吞吐且灵活的主机网络
authors: "Athinagoras Skiadopoulos, Zhiqiang Xie, Mark Zhao, Qizhe Cai, Saksham Agarwal, Jacob Adelmann, David Ahern, Carlo Contavalli, Michael Goldflam, Vitaly Mayatskikh, Raghu Raja, Daniel Walton, Rachit Agarwal, Shrijeet Mukherjee, Christos Kozyrakis"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi24-skiadopoulos.pdf"
tags: ["query:dcn-comm"]
score: 7.0
evidence: 面向加速计算的网卡与主机网络栈协同设计
tldr: 现有主机网络栈在吞吐与灵活性之间被迫取舍，难以满足GPU加速AI应用的带宽需求。本文采用网卡硬件与软件栈协同设计的全新方案，将数据路径与控制路径物理分离，使数据传输高效而传输层决策可灵活定制。基于原型实现，系统在保持接近硬件线速吞吐的同时支持多种协议扩展，为数据中心高性能通信基础设施提供了新架构。
source: OSDI-2024-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-high-throughput-and-flexible-host-networking-for-accelerated-computing/fig-001.webp\", \"caption\": \"\", \"page\": 1, \"index\": 1, \"width\": 875, \"height\": 356}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-high-throughput-and-flexible-host-networking-for-accelerated-computing/fig-002.webp\", \"caption\": \"\", \"page\": 1, \"index\": 2, \"width\": 875, \"height\": 360}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-high-throughput-and-flexible-host-networking-for-accelerated-computing/fig-003.webp\", \"caption\": \"\", \"page\": 5, \"index\": 3, \"width\": 490, \"height\": 364}]"
motivation: 现有主机网络栈难以兼顾高吞吐与协议灵活性，制约GPU加速AI应用的网络带宽发挥。
method: 论文提出网卡硬件与软件栈协同设计，将数据路径与控制路径物理分离以兼顾性能与灵活。
result: 原型系统在接近硬件线速吞吐的同时支持可定制的传输协议，验证了设计可行性。
conclusion: 该工作为数据中心高性能、可扩展的主机网络架构提供了新范式。
---

## Abstract
Modern network hardware is able to meet the stringent bandwidth demands of applications like GPU-accelerated AI. However, existing host network stacks offer a hard tradeoff between performance (in terms of sustained throughput when compared to network hardware capacity) and flexibility (in terms of the ability to select, customize, and extend different network protocols). This paper explores a clean-slate approach to simultaneously offer high performance and flexibility. We present a co-design of the NIC hardware and the software stack to achieve this. The key idea in our design is the physical separation of the data path (payload transfer between network and application buffers) and the control path (header processing and transport-layer decisions). The NIC enables a high-performance zero-copy data path, independent of the placement of the application (CPU, GPU, FPGA, or other accelerators). The software stack provides a flexible control path by enabling the integration of any network protocol, executing in any environment (in the kernel, in user space, or in an accelerator). We implement and evaluate ZeroNIC, a prototype that combines an FPGA-based NIC with a software stack that integrates the Linux TCP protocol. We demonstrate that ZeroNIC achieves RDMA-like throughput while maintaining the benefits of robust protocols like TCP under various network perturbations. For instance, ZeroNIC enables a single TCP flow to saturate a 100Gbps link while utilizing only 17% of a single CPU core. ZeroNIC improves NCCL and Redis throughput by 2.66X and 3.71X, respectively, over Linux TCP on a Mellanox ConnectX-6 NIC, without requiring application modifications.

---

## 论文详细总结（自动生成）

# 论文总结：High-throughput and Flexible Host Networking for Accelerated Computing（ZeroNIC，OSDI 2024）

## 1. 核心问题与研究动机

- **背景**：GPU 加速的 AI 训练、数据分析、分布式存储等应用对主机网络带宽需求急剧上升（如 NVIDIA DGX-B200 单机 3.2 Tbps，每 GPU 400 Gbps），端主机网络栈正成为主要瓶颈。
- **核心矛盾**：现有主机网络栈在**性能**与**灵活性**之间被迫二选一：
  - **RDMA 路线（InfiniBand / RoCE）**：吞吐高，但协议固化在网卡硬件/固件中，几乎不可定制；依赖无损网络、PFC、专用交换机和布线，面对丢包、乱序、拥塞时脆弱（RoCE 在 1% 丢包下吞吐可能崩塌），且修改协议需网卡厂商介入。
  - **Linux 内核 TCP 路线**：协议丰富、健壮、易扩展，但单流性能受 CPU 限制。论文实测：即便开启 TSO/GRO、9K MTU、发送端零拷贝，单条 TCP 流在 Mellanox ConnectX-6 上最高仅约 50 Gbps，且接收端协议处理线程已 100% 占满。
  - **关键归因**：接收侧从内核缓冲区到用户缓冲区的**数据拷贝**是主要开销；论文用“截断 payload 模拟 RX 零拷贝”的实验证明，去掉该拷贝即可打满 100 Gbps。
- **论文含义**：提出一种“clean-slate”的网卡硬件与软件栈**协同设计**，试图同时获得 RDMA 级吞吐与 Linux 协议栈级灵活性，并适配 CPU、GPU、FPGA 等任意加速器内存。

## 2. 方法论

### 2.1 核心思想

- **数据路径与控制路径物理分离、逻辑耦合**：
  - **数据路径**：网卡负责在网口与应用缓冲区（CPU DRAM、GPU HBM、FPGA 等）之间做**零拷贝 DMA**，只搬运 payload。
  - **控制路径**：由软件栈承载任意传输协议（内核、用户态或加速器内），只处理**包头与传输层决策**（拥塞控制、流控、确认、保序交付等）。
  - 分离后仍通过“逻辑耦合”保证协议语义：数据先落到正确位置，但只有在协议确认后才对应用可见。

### 2.2 网卡硬件设计

- **数据结构**（图 7）：
  - **MR Table**：记录应用注册的 Memory Region（连续虚拟地址区间），经网卡内 IOMMU 做地址翻译（支持 CPU→PCIe→GPU 显存映射，类似 GPUDirect）。
  - **MS List**：每个发送/接收操作对应一个 Memory Segment（MR ID + 偏移 + 长度），按流组织为链表，描述待填/待发缓冲区。
  - **Flow Table**：以五元组（源/目的地址、端口、协议号）索引，保存 flow ID 与 **flow cursor**（最后一个按序消费的包序号）。
- **收包决策流程**（图 8），依据包序号与 flow cursor 的关系分五种情况：
  - **(a) 按序新数据**：接受，依据 MS List 与 MR 信息计算应用地址并零拷贝 DMA，推进 cursor，回收已完全消费的 MS。
  - **(b) 完全重复**：丢弃。
  - **(c) 部分重复**：截掉重复前缀（trim），其余按 (a) 接受。
  - **(d) 乱序/有空洞**：仍接受并 DMA 到对应 MS（通过累加 MS 长度定位），但不回收 MS，直到 cursor 越过；控制栈周期性把最新确认字节同步给网卡以更新 cursor。
  - **(e) 无匹配 MS 或流表满**：丢弃、暂存网卡内存，或**退化到非零拷贝慢路径**（原型采用后者）。
- **发包路径**：按请求 FIFO 顺序，从应用缓冲区 DMA 取数据，与软件下发的包头在 Split/Merge 单元合并成包，可选做 TSO 分段。
- **可扩展性**：不为乱序包缓存 payload（直接 DMA 到正确位置），显著降低网卡内存需求；10K 高性能流约需 10 MB（Flow Table ~700 KB、MR Table ~100 KB、MS List 采用 CIR/PIR 式按需分配，约 9.77 MB）。

### 2.3 控制栈与 API

- **两套队列**：
  - **应用队列**：与 provider library 共享内存，含 SQ/RQ/CQ，轮询避免系统调用与中断。
  - **网卡队列**：控制栈与网卡之间，承载 MS、包头、完成事件，由网卡 DMA 访问。
  - 传输协议被“夹”在两组队列之间，从而可支持任意协议与任意执行环境。
- **正确性机制**：
  - 发送侧重传：控制栈克隆 socket buffer 元数据直至收到 ACK，硬件重建相同 MS 以支持重传。
  - 接收侧：只回收 cursor 之后的 MS，允许未来重传数据覆盖已 DMA 的位置，简化“新旧数据混合”处理。
  - 每流独立分配网卡队列与 flow ID，避免多流复用带来的跟踪复杂度。
- **消息语义支持**：在 TCP options 字段中加入**消息序号**，与流序号配合定位 MS，从而在流式协议之上支持 RDMA verbs 式消息语义。
- **API 兼容**：实现 `libibverbs`（动态替换 `verbs_context_ops`）与类 socket 流式 API，应用无需修改；兼容 TSO/GRO/LRO 等卸载机制。

## 3. 实验设计

- **硬件平台**：
  - ZeroNIC 原型：Xilinx Virtex UltraScale+ FPGA 的 100 Gbps 以太网网卡，3 条 x16 PCIe 3.0（1 条连 CPU、2 条连 GPU），本质上是 CPU 与两块 GPU 之间的 PCIe 交换机；CPU 侧用 QDMA IP，GPU 侧用 XDMA IP。
  - 主机：32 核 AMD EPYC 7502，L1/L2/L3 = 2 MB/16 MB/128 MB；GPU 为 NVIDIA Quadro RTX 4000。
  - 两节点直连 100G 以太网。
  - 原型 IP 限制：GPU→FPGA 最高 85.0 Gbps，FPGA→GPU 最高 38.6 Gbps（非设计固有，ASIC 可消除）。
- **对比基线**（均用 Mellanox ConnectX-6）：
  - **MLX TCP**：Linux 内核 TCP，开/关发送端零拷贝（TX ZC），TSO/LRO/9K MTU。
  - **MLX RoCE**：RoCEv2 RC，4K MTU。
  - ZeroNIC 配置：内核 TCP（BBR）+ TSO/GRO + 9K MTU。
- **实验组**：
  1. **单流 CPU–CPU 吞吐与接收端 CPU 分解**（表 2）：96.37 Gbps、sys 仅 17.20%，对比 TCP 50.63 Gbps@100% CPU 与 RoCE 98.03 Gbps。
  2. **MTU 扫描**（图 9）：1500/4000/9000 字节，及 RoCE 的 1024/4096。
  3. **跨端点吞吐**（表 3）：CPU–CPU、CPU–GPU、GPU–GPU，并据 CPU 利用率外推单流上限（>500 Gbps，跨端点 >400 Gbps）。
  4. **NCCL broadcast 基准**（图 10）：4 KB–64 MB 集合通信大小，对比“同一 FPGA 网卡但全包走内核栈”的 TCP 基线，大消息下 2.66× 提升。
  5. **PyTorch DDP 训练**（表 4）：ResNet50/101/152，每 epoch 100 iteration，对比 RoCE + GPUDirect，延迟差距在 2% 以内。
  6. **Redis**（图 11）：`redis-benchmark` SET 操作，1 KB–1 MB，4 客户端 × 10 未完成请求，平均达 RoCE 的 89%，为 MLX TCP 的 3.71×。
  7. **多流 incast 鲁棒性**（图 12）：2 MB 零拷贝长流（1–8 条）与 64 KB 非零拷贝短流（1–8 条）混合，考察吞吐、CPU 利用率与短流 p50 时延。
  8. **注入丢包**（图 13）：0.1%–10% 概率丢包，GPU–GPU 单流对比 MLX TCP（CPU–CPU）；RoCE 因缺少可拦截 RDMA 流量的丢包规则未能对比。

## 4. 资源与算力

- 论文属于**网络系统与硬件原型**研究，**未涉及大模型训练算力消耗**，也未报告训练时长、GPU 卡数规模等指标。
- 明确给出的资源仅为测试床：1 块 FPGA 网卡、1 颗 32 核 AMD EPYC 7502、2 块 NVIDIA Quadro RTX 4000、两节点直连；PyTorch 实验为 ResNet 系列、每 epoch 100 次迭代，但未说明总 epoch 数与总训练时长。
- 未给出功耗、成本、芯片面积等硬件代价数据。

## 5. 实验数量与充分性

- **规模**：约 8 组实验，覆盖微基准（吞吐、MTU、跨端点）、真实应用（NCCL、PyTorch、Redis）、鲁棒性（多流冲突、丢包），整体结构较完整。
- **充分性**：
  - 优点：同时在 CPU 与 GPU 内存路径上验证；既有线速微基准又有端到端应用；显式做了丢包与多流干扰实验，直接回应 RDMA 的痛点；给出 CPU 利用率分解与外推上限，便于判断瓶颈归属。
  - 不足：
    - 仅**两节点直连**拓扑，无交换机、无多跳、无大规模集群（论文以万卡/千节点场景为动机，但未验证）。
    - 仅有 **TCP 一种协议**的实测，“任意协议可插拔”只停留在设计论证。
    - 受 FPGA IP 限制，GPU–GPU 仅 38.6 Gbps，未真正展示 GPU 场景下的线速；ASIC 性能为外推而非实测。
    - 零拷贝流数量受硬件队列限制（原型仅 8 条），与“10K 流”的资源分析之间存在实测缺口。
    - 未与 SRNIC、IRN、Flor、1RMA 等学术方案做直接实验对比，仅文字讨论。
    - 丢包实验未能覆盖 RoCE 基线，削弱了“RoCE 在丢包下崩塌”的直接对照力度。
- **公平性**：基线调优较认真（TSO/LRO、9K MTU、CPU 绑核、同物理核超线程、开启发送零拷贝），对比口径基本合理；但 ZeroNIC 使用定制 FPGA 网卡而基线使用商用 ConnectX-6，硬件代际与实现成熟度不完全对等，属于原型研究常见但需注意的偏差。

## 6. 主要结论与发现

- 数据/控制路径物理分离可在**不牺牲协议灵活性**的前提下达到 RDMA 级吞吐：单条 TCP 流 96.37 Gbps，仅占 17.2% 单超线程；基线 TCP 50.63 Gbps 即打满 CPU。
- 按 CPU 利用率外推，ZeroNIC 单流可达 **>500 Gbps（CPU–CPU）/ >400 Gbps（含 GPU 路径）**，约为现有 Linux TCP 单流上限的 11×；8 GPU、3.2 Tbps 规模仅需 8 个 CPU 超线程。
- 零拷贝收益可延伸到加速器：CPU–GPU 达 84.78 Gbps、GPU–GPU 达 38.59 Gbps，均达到原型硬件上限。
- 应用无需修改即可受益：NCCL 提升 2.66×、Redis 提升 3.71×，PyTorch 训练延迟与 RoCE + GPUDirect 差距在 2% 以内。
- 保留 TCP 健壮性：1% 丢包下仍接近满吞吐；多流 incast 下带宽分配公平、总 CPU 占用基本恒定、短流时延不受长流数量影响。
- 结论性主张：这是首个在**收发两侧**同时支持可靠协议（如 TCP）零拷贝、且不要求 MTU 对齐或修改 API 的工作。

## 7. 优点

- **问题切分清晰**：“物理分离 + 逻辑耦合”的抽象简洁有力，把“性能”归给硬件数据面、把“灵活性”归给软件控制面，避免了以往二选一的架构性妥协。
- **硬件设计克制**：不缓存乱序 payload，而是靠 MS List + flow cursor 直接 DMA 到最终位置，显著降低网卡 SRAM 需求（10K 流约 10 MB），并给出 CIR/PIR 式按需分配方案。
- **兼容性出色**：复用未修改的 Linux TCP 与 `libibverbs`，NCCL/PyTorch/Redis 零改动运行，落地阻力小；同时兼容 TSO/GRO/LRO。
- **设备无关性**：数据路径对 CPU DRAM、GPU HBM、FPGA 一视同仁，契合加速计算趋势，也为未来 CXL/NVLink 留出空间。
- **正确性考虑细致**：重传、部分重复、乱序、流表满、消息边界、MS 回收时机等边界条件均有明确处理策略。
- **评估维度较全**：微基准 + 真实应用 + 故障注入，且给出 CPU 分解与外推，论证链条较完整。

## 8. 不足与局限

- **原型能力受限**：FPGA + XDMA IP 使 GPU 方向带宽被压到 38.6/85 Gbps，GPU–GPU 线速与 ASIC 可行性均为推断，缺少芯片级实现或仿真支撑。
- **规模验证不足**：仅两节点直连，无交换机、无拥塞场景、无千节点级集合通信；而论文动机恰恰来自万卡 AI 集群。
- **协议多样性未实证**：只用内核 TCP，用户态协议栈、加速器内协议终止等“任意环境”主张未做实验；消息序号依赖 TCP options，属于对标准 TCP 的私有扩展，跨厂商互通性存疑。
- **流

- **流级可扩展性依赖硬件资源线性增长**：每流独立分配网卡队列与 flow ID 的设计简化了跟踪逻辑，但也意味着 Flow Table、MS List、队列上下文随并发流数线性膨胀；论文给出的“10K 流约 10 MB”只是容量估算，未实测 10K 流下的吞吐、时延与表项冲突行为，也未量化 flow table 满时退化到非零拷贝慢路径的性能损失幅度。原型仅 8 条零拷贝流，与目标场景（万卡集群中每 GPU 数百条流）之间存在明显的实测缺口。
- **对既有生态的隐性依赖**：方案实质要求网卡具备 IOMMU 式地址翻译、可编程 DMA 描述符、以及 CPU/GPU 双上行端口（原型即充当 PCIe switch）。这与当前商用网卡（ConnectX 系列）的硬件架构差异较大，短期内难以在现成硬件上复现；论文亦未讨论在多租户/虚拟化环境下，MR Table 与地址翻译带来的隔离与安全问题（如恶意 MS 越界访问、IOMMU 粒度与页表一致性）。
- **消息语义扩展的互通风险**：借助 TCP options 携带消息序号来实现 verbs 式消息边界，属于对标准 TCP 的私有扩展。虽然作者声称不影响中间件与卸载，但在经过 middlebox、负载均衡器、或其他厂商网卡时，该字段的保留/剥离行为未被验证；同时 options 空间有限（40 字节），未来若要携带更多元数据（如多路径标识、租户 ID）将受限。
- **单向语义与完整 RDMA 能力未覆盖**：实验以流式收发（send/recv）与集合通信为主，对 one-sided write/read、原子操作、内存注册语义、以及 RDMA 特有的完成事件模型仅做了 API 兼容性声明，缺少针对性微基准（如 one-sided 小消息 IOPS、原子操作吞吐）。
- **时延与尾延迟刻画不足**：全文以吞吐与 CPU 占用为核心指标，仅在 incast 实验中报告了短流 p50 时延，缺少 p99/p999 尾延迟、空载往返时延（RTT）、以及小消息（<1 KB）请求-响应模式的系统测量。对 Redis 这类延迟敏感负载，89% RoCE 的平均值结论尚不足以说明尾延迟表现。
- **故障与异常路径验证有限**：正确性论证以设计推演为主（重传、部分重复、乱序、流表满），缺少针对性的压力测试或形式化验证；丢包实验最高 10%，但未覆盖乱序注入、位错、链路抖动、网卡复位等更接近真实数据中心故障的场景。
- **功耗、面积与成本未评估**：作为硬件协同设计工作，缺少 FPGA 资源占用（LUT/BRAM）、ASIC 面积估算、功耗对比（相对 ConnectX-6 的每 Gbps 功耗），使得“可落地性”的论证停留在功能层面。
- **控制面 CPU 开销并非为零**：17.2% 的单超线程占用虽远优于内核 TCP 的 100%，但在 400 Gbps/800 Gbps 目标速率下，控制栈的每包处理成本（含 ACK 处理、拥塞控制、MS 管理）是否仍能维持在单核量级，论文仅以线性外推给出结论，缺少高速率下的实测支撑。
- **未与学术同类方案直接对比**：与 SRNIC、IRN、Flor、1RMA、以及可编程网卡（如 Tofino/NanoPU/Pluto）相关工作仅有文字讨论，没有同台实验，读者难以判断 ZeroNIC 相对已有“软硬协同”路线的净收益。

## 9. 可迁移的设计经验与未来方向

- **抽象层面的启示**：“数据面归硬件、控制面归软件、以 cursor/序号做逻辑耦合”这一模式并不限于 TCP——它可推广到 QUIC、自定义数据中心传输协议、乃至存储协议（NVMe-oF）与集合通信库自带的可靠传输，为“协议可编程”提供了一条不依赖可编程交换机/可编程流水线的路径。
- **内存语义的通用化**：将 MR/MS 抽象建立在“设备无关的应用缓冲区”之上，天然适配 CPU DRAM、GPU HBM、FPGA、以及未来的 CXL 内存池，这为“内存为中心”的数据中心网络提供了接口雏形。
- **潜在后续工作**：(1) 在 ASIC 或量产可编程网卡上实现并做线速验证；(2) 在多跳、带交换机的集群中评估拥塞控制交互与 incast 行为；(3) 支持用户态协议栈与加速器内协议终止；(4) 引入多路径与包喷洒，配合网卡按序放置能力；(5) 补齐安全隔离（per-MR 权限、IOMMU 粒度、虚拟机直通）与尾延迟评估；(6) 与 SRNIC 等方案做同硬件平台的对照实验。

## 10. 总体评价

- 论文的核心贡献不在于某项单点优化，而在于**重新划分了网卡与主机软件之间的职责边界**：把“搬数据”彻底交给硬件、把“做决策”彻底留给软件，并用 MS List + flow cursor 这一轻量机制解决了零拷贝与乱序、重传、消息语义之间的耦合难题。其价值在于同时化解了 RDMA 的僵化与内核 TCP 的低效这一长期两难，且以“应用零改动”为约束，落地路径清晰。
- 从证据强度看，论文在**单流吞吐与 CPU 开销**这一关键命题上论证扎实（96.37 Gbps @ 17.2% CPU，并有 CPU 分解与 MTU 扫描佐证），在应用层（NCCL 2.66×、Redis 3.71×、PyTorch 差距 <2%）也给出了有说服力的端到端结果，丢包与 incast 实验则有效回应了对 RDMA 脆弱性的质疑。
- 但从**系统级说服力**看，仍存在三处关键跳跃：由 FPGA 原型外推至 ASIC 线速、由两节点直连外推至万卡集群、由单一 TCP 外推至“任意协议”。这些跳跃在原型论文中可被接受，但也正是后续工作必须补齐的部分。
- 综合判断：这是一篇**架构清晰、工程完成度高、评估维度较全**的系统论文，其“物理分离 + 逻辑耦合”的设计原则具有较强的一般性与启发性；其实验覆盖的广度优于深度，结论在单机单流维度可信，在集群与多协议维度尚属待验证的合理假设。

（完）
