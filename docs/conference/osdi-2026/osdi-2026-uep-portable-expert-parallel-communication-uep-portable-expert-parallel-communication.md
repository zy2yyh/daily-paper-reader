---
title: "{UEP}: Portable {Expert-Parallel} Communication"
title_zh: UEP：可移植的专家并行通信
authors: "Ziming Mao, Yihan Zhang, Chihan Cui, Zhen Huang, Kaichao You, Zhongjie Chen, Zhiying Xu, Zhenyu Gu, Scott Shenker, Costin Raiciu, Yang Zhou, Ion Stoica"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-mao-ziming-uep.pdf"
tags: ["query:dcn-comm"]
score: 8.0
evidence: 以CPU代理替代GPU发起RDMA的可移植专家并行通信
tldr: MoE训练依赖专家并行通信，而DeepEP等库基于GPU发起的RDMA，需要GPU与网卡紧耦合，导致在异构硬件上可移植性差。作者提出UEP，用高吞吐的GPU-CPU控制通道把紧凑的令牌路由命令交给多线程CPU代理，再由代理驱动网络传输。实验表明UEP在多种GPU与网卡组合上都能保持接近专用方案的高性能。工作为分布式训练中的集合式通信提供了可移植的高效实现路径。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2026-usenix/osdi-2026-uep-portable-expert-parallel-communication/fig-001.webp\", \"caption\": \"\", \"page\": 7, \"index\": 1, \"width\": 704, \"height\": 406}]"
motivation: 现有专家并行通信库依赖GPU发起RDMA，需要GPU与网卡深度耦合，在异构硬件上可移植性差。
method: UEP用高吞吐GPU-CPU控制通道将令牌路由命令转交多线程CPU代理，由代理完成网络通信。
result: 在多种异构GPU与网卡组合上，UEP保持了接近专用方案的高吞吐通信性能。
conclusion: 表明将通信发起从GPU卸载到CPU代理可在不牺牲性能的前提下获得硬件可移植性。
---

## Abstract
Modern Mixture-of-Experts (MoE) workloads rely on expert parallelism (EP) to achieve high GPU efficiency. State-of-the-art EP communication libraries, such as DeepEP, rely on GPU-initiated RDMA communication. Although performant, they have poor portability across heterogeneous GPU and NIC hardware. The poor portability is rooted in its architecture: GPU-initiated RDMA communication requires tight vertical integration between GPUs and NICs, e.g., GPU writing to NIC driver/MMIO interfaces. We present UEP, a portable EP communication system that delivers high performance across heterogeneous GPU and NIC hardware. UEP replaces GPU-initiated RDMA with a high-throughput GPU-CPU control channel: compact token-routing commands are transferred to multithreaded CPU proxies, which then issue GPUDirect RDMA operations on behalf of GPUs. UEP further emulates various ordering semantics required by specialized EP communication modes using RDMA immediate data, enabling correctness on NICs that lack such ordering, e.g., AWS EFA. We implement UEP on NVIDIA and AMD GPUs with EFA and Broadcom NICs. On EFA, it outperforms the best existing EP solution by 2.1× for dispatch and combine throughput. UEP also improves token throughput on SGLang by up to 40% on the NVIDIA+EFA platform, and improves DeepSeek-V3 training throughput over the AMD Primus/Megatron-LM framework by up to 45% on a 16-node AMD+Broadcom platform.

---

## 论文详细总结（自动生成）

# UEP：可移植的专家并行通信（OSDI 2026）论文总结

## 一、核心问题与整体含义（研究动机与背景）

- **背景**：MoE 已成为 DeepSeek-V3、gpt-oss、Gemini-3 Pro、Llama 4 等前沿 LLM 的主流架构。专家并行（EP）把不同专家放置在不同 GPU 上，以 all-to-all 方式交换 token 激活，是释放 GPU 显存、支撑大 batch 的关键手段。
- **EP 通信的独特性**：token 激活很小（约 7KB，FP8/7168 隐层），路由目标在运行时由 GPU 上的门控网络决定，消息高度不规则、负载不均衡；通信已是瓶颈——可占前向 43.6%、端到端训练 32%、推理 47%。
- **现有方案**：DeepEP 等 SOTA 库采用 **GPU 发起的 token 级 RDMA（IBGDA）**，GPU 线程直接向网卡提交传输命令，实现细粒度流水线重叠（与拷贝、计算、去重、分层 reduce 重叠），性能优异。
- **核心矛盾（可移植性差）**：
  - *硬件互操作*：IBGDA 要求 GPU 直接写 NIC 驱动定义的 MMIO 接口，需 GPU 与 NIC 厂商级紧耦合与专有寄存器布局，非 NVIDIA GPU 或非 IB 网卡基本无法支持。
  - *语义假设*：GPU 内核假设网络提供严格顺序（如 write-then-atomic），而 AWS EFA（SRD）等云网卡只保证可靠、不保证有序，直接破坏正确性。
  - *适配成本*：m 种 GPU × n 种网卡需 O(m×n) 的垂直集成工作量，形成厂商锁定。DeepEP 目前无法跑在 AWS GPU+EFA、AMD+Broadcom 等主流组合上。
- **论文含义**：提出 UEP——首个在非 NVIDIA 平台上实现 GPU 发起 token 级 EP 通信的系统，把适配成本降为 O(m)，在不牺牲性能的前提下获得跨异构 GPU/NIC 的可移植性。

## 二、方法论

### 2.1 核心思想：解耦"通信发起"与"通信执行"

- GPU 仍负责**发起**细粒度 token 级通信（保留与计算/拷贝流水线重叠的能力），但把**执行与管理**委托给宿主 CPU 代理。
- GPU 线程通过 PCIe 向 CPU 发送紧凑控制命令（仅含目的端、偏移、长度、序号等），数据载荷仍留在 GPU 显存；CPU 代理代为发起 **GPUDirect RDMA**。
- CPU 经 libibverbs 可移植到任意网卡、经 CUDA/ROCm 可移植到任意 GPU，且灵活实现各类交付语义。

### 2.2 高效 GPU-CPU 通信通道

- **128 位 TransferCmd**：单条 GPU 指令 + 一次 MMIO doorbell 即可写入，对齐 PCIe 事务与载荷粒度。
- **无锁 FIFO**：GPU 为生产者（推进 head），CPU 代理为消费者（推进 tail）；head 放 GPU 内存、tail 放 host 内存，避免任一侧反复跨 PCIe 轮询。
- **限流**：以 `kMaxInflight` 约束在途命令数，FIFO 满即阻塞 GPU，从而对 GPU 发送速率施加反压。
- **多通道降竞争**：每 GPU 多条 FIFO，按需把要求保序的命令映射到同一 FIFO；跨 FIFO 不保证顺序。
- **内存一致性**：GPU 侧绕过硬件缓存，CPU 侧刷新写至 host 内存（或借助 GH200 的 C2C 缓存一致性）。
- **API**：CPU 侧 `Poll`（读取不弹出）/`Pop`（消费）；GPU 侧 `Push`（入队返回 Idx）/`Check-completion(Idx)`（查询是否已被消费）。
- **四类命令**：`Write`（可捎带 atomic 通知完成）、`Atomics`（远程门铃/计数器）、`Drain`（排空未完成 RDMA）、`Barrier`（全 peer 分层 barrier 或同 rail barrier）。

### 2.3 多线程 CPU 代理

- 每 GPU 一个代理，最多 4 个绑定核的线程；第 i 线程只与远端第 i 线程建连，限制连接数。
- 维护**对称内存**语义：仅传 27 位偏移而非 64 位地址，节省命令位宽，并摆脱 NVSHMEM/rocSHMEM 依赖。
- 线程按 NUMA 管理同组多网卡；LL 模式每目的 rank 一个 QP，HT 模式每通信通道多个 QP，并可跨不同带宽网卡聚合带宽。

### 2.4 用 CPU 代理模拟交付语义（关键创新）

- 利用 **RDMA immediate data**（32 位，已标准化进 RoCEv2 包头）携带序列号/expert 索引，接收端 CPU 代理解析后在 control buffer 中暂存 atomic，条件满足后再应用。
- **LL 模式（partial completion fence）**：atomic 需等待"发往指定 expert 的 X 个 write 已到达"才生效；不要求这些 write 有序，也不受其他 expert 的写影响。
- **HT 模式（partial ordering）**：按通信通道（ring buffer）局部保序；乱序到达的 atomic 暂存，待前序消息应用后依序应用，避免接收端读到陈旧槽位或发送端覆盖未读数据。
- **在接收端而非发送端强制顺序**：发送端方案需多等一个 RTT（图 7 实测更差）；接收端方案几乎零额外开销。
- **最低硬件要求**：RDMA 可靠传输 + 小控制元数据（immediate data），主流 RDMA 网卡均满足。

### 2.5 实现要点

- 基于 DeepEP 扩展：20.8K 行 C++（含 2.4K CUDA/ROCm）+ 1K 行 Python，**API 兼容、零代码改动替换**。
- **EFA**：无硬件原子操作 → 软件原子（write + 携带计数的 immediate write，CPU 更新 host 内存计数器）。
- **AMD**：PTX→ROCm 内建、warp→wavefront（WARP_SIZE 32→64）、TMA→CU 拷贝、wavefront 特化（HT 模式合并协调者与接收者 wavefront）。
- 移植到 AMD+EFA 仅需 3 人月。

## 三、实验设计

### 3.1 测试平台（表 2，均为公有云租用）

| 平台 | 规模 | GPU | 网络 |
|---|---|---|---|
| NV_EFA3 | 4 节点 | H200×8 | AWS EFAv3 200G×16 |
| NV_EFA4 | 4 节点 | B200×8 | AWS EFAv4 400G×8 |
| NV_IB | 4 节点 | H100×8 | ConnectX-7 400G（Nebius） |
| NV_C2C_IB | 2 节点 | GH200×1 | CX7 200G（Lambda） |
| AMD_IB | 4–16 节点 | MI300X×8 | CX7 400G（OCI） |
| AMD_BRC | 4 节点 | MI300X×8 | Broadcom Thor-2 400G（Vultr） |

### 3.2 对比基线

- NCCL/RCCL（CPU 发起集合通信）、DeepEP（GPU 发起 SOTA）、PPLX（Perplexity 定制 MoE 内核）、CPU-assisted IBGDA（用单线程代理模拟）、Mori（ROCm 原生 RDMA 库）、理论带宽上限。

### 3.3 实验场景

- **应用级训练**：AMD Primus/Megatron-LM 训练 DeepSeek-V3（缩减为 32 层、379B），16 节点，EP8/16/32 + PP16/8/4，报告 TFLOPS/GPU 与 tokens/s。
- **应用级训练**：H200+EFA 上 Megatron-LM，Qwen3-235B（长序列 seq=49152；大批次 seq=20480, GBS=128）与 DeepSeek-V3（12 层约 135B），EP=16/32。
- **应用级推理**：SGLang v0.5.3，DeepSeek-R1-0528 与 Qwen3-235B-A22B-FP8，EP=16/32，prefill 为主（输入 4096/输出 5）。
- **微基准**：EP32 下 dispatch/combine 延迟随 token 数变化（EFA3、EFA4、IB、GH200、AMD_IB、AMD_BRC）。
- **设计剖析**：FIFO 压力测试（1–8 Mops，8 条 FIFO 并发）、EP 规模敏感性（16→32）、CPU 线程数（1/2/4）、模拟原子 vs 原生原子 vs 纯 write 的延迟（8B–4KiB）。

## 四、资源与算力

- **已明确披露**：各测试床的 GPU 型号与数量（H200/B200/H100/GH200/MI300X，单实验最多 16 节点×8 = 128 卡）、CPU 核数、网卡规格；全部为公有云租用实例。
- **未披露**：总 GPU 小时、训练/评测总时长、能耗与租用成本
