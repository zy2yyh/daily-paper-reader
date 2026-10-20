---
title: "{SBB}: Eliminating Centralized Bottlenecks in Userspace Network Runtime"
title_zh: SBB：消除用户态网络运行时中的集中式瓶颈
authors: "Kang Hu, Shuqi Dong, Chuandong Li, Ran Yi, Zonghao Zhang, Yiming Yao, Bo An, Jie Zhang, Xiaolin Wang, Yingwei Luo, Zhenlin Wang, Diyu Zhou"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-hu-kang.pdf"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 面向高吞吐低时延的去中心化用户态网络运行时调度
tldr: 用户态网络运行时为兼顾吞吐、时延与CPU效率需做请求抢占、CPU分配和核间负载均衡，但既有设计依赖集中式组件，随工作核增多成为扩展瓶颈。SBB提出完全去中心化的运行时，在系统机制和调度策略上同时改进。评测显示其在保持高吞吐与低尾时延的同时实现高CPU效率和近似线性扩展。该设计为数据中心网络栈的调度可扩展性提供了新路径。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 用户态网络运行时需同时完成请求抢占、CPU分配与核间负载均衡，但集中式组件随工作核数增加成为性能扩展瓶颈。
method: 设计完全去中心化的用户态网络运行时，在系统机制与调度策略两方面消除集中式协调环节。
result: 在高并发下同时保持高吞吐、低时延与高CPU效率，性能随工作核数近似线性扩展。
conclusion: 去中心化调度可替代集中式设计，为高吞吐低时延的数据中心网络栈提供可扩展基础。
---

## Abstract
To achieve high throughput, low latency, and high CPU efficiency, userspace network runtimes must perform three types of scheduling: 1) request preemption to minimize tail latency, 2) CPU allocation among services to avoid wasting CPU cycles when the load is low, and 3) request load balancing across worker cores to achieve work conservation. However, prior designs rely on centralized components, which inevitably become scalability bottlenecks as the number of CPU workers increases, limiting system performance scaling. This paper presents SBB, a purely decentralized userspace network runtime that simultaneously delivers high performance, high CPU efficiency, and high scalability by advancing in both system mechanism and scheduling policy. For system mechanism, SBB leverages the emerging User Interrupt mechanism in a novel way, delivering two types of device interrupts to the userspace runtime: 1) user-level timer interrupts for request preemption, and 2) user-level NIC interrupts for packet arrival to perform CPU allocation. For scheduling policy, SBB introduces a two-level algorithm, which marries flow migration (for persistent imbalance) with task stealing (for temporary imbalance), challenging the conventional wisdom that centralized load balancing outperforms decentralized approaches. Evaluation shows that SBB achieves 1.7× to 5.2× higher throughput when scaling to 48 cores, while meeting the same tail latency target.

---

## 论文详细总结（自动生成）

# SBB：消除用户态网络运行时中的集中式瓶颈 —— 论文深度总结

## 1. 核心问题与研究动机

- **背景**：现代数据中心网卡带宽已达数百 Gbps，而单核乃至十余核 CPU 难以匹配网卡速率，因此用户态网络运行时（kernel-bypass）必须依靠多核扩展。同时系统需同时满足三项指标：高吞吐、低（尾）时延、高 CPU 效率。
- **三类必需调度**：
  1. **请求抢占（request preemption）**：防止长请求造成短请求的队头阻塞，压低 p99.9 尾时延；
  2. **CPU 分配（CPU allocation）**：LC（延迟敏感）与 BE（尽力而为）应用共置时，低负载下把空闲核让给 BE，突发时迅速收回；
  3. **请求负载均衡**：跨 worker 核调度请求，实现 work conservation。
- **核心痛点**：既有系统（Shinjuku、Concord、Caladan、Skyloft、TQ 等）为完成上述调度，均依赖**集中式实体**——集中式 timer（抢占）、集中式 monitor/iokernel（CPU 分配）、集中式 dispatcher（负载均衡）。这些实体随 worker 数增加必然成为扩展瓶颈。
- **论文实测揭示的瓶颈证据**（Fig.3）：
  - Shinjuku 的集中 timer 在 worker 超过 16 后无法在 5 μs 抢占间隔内发完 IPI，吞吐停滞甚至下降；
  - Caladan 的 iokernel 在 on-off 突发负载下无法及时回收核，SLO 违约；
  - 集中 dispatcher 使系统吞吐被"最窄的上游"卡死在约 5 MRPS（单包处理约 200 ns）；
  - **"多 dispatcher 分组"这一常见假设被证伪**：分组越多，负载不均衡重现，平均吞吐反而下降；
  - 用户态 IPI（UIPI）虽有性能优势，但受单 NUMA 约束，发送速率有上限，增加发送核也无法扩展。
- **被挑战的"常识"**：集中式负载均衡更接近排队论证明最优的共享 FCFS 队列（JBSQ），因此被认为优于去中心化方案。论文指出该观点忽略了可扩展性约束，并用仿真证明**去中心化策略在质量上可以逼近集中式**。

## 2. 方法论

### 2.1 核心思想
构建**纯去中心化**的用户态网络运行时：每个 worker 线程绑定一个核，独占自己的本地队列、本地 timer、专属 Rx/Tx 队列，全局不存在跨 worker 的共享组件；同时不牺牲三类调度的能力与质量。

### 2.2 系统机制：以 User Interrupt（UINTR）投递两类"设备中断"到用户态

UINTR 是 Intel Sapphire Rapids 引入的硬件特性，允许用户态直接收发中断而无需陷入内核。已有工作（Aspen-KB、Libpreemptible、Junction、Vessel、Skyloft 等）几乎只用它做 UIPI；**SBB 首次将其用于 NIC 与 timer 两类设备中断**（该用法未见于 Intel 硬件手册）。

- **NIC UINTR → 实现及时 CPU 分配（替代轮询）**
  - 根因分析：轮询与多应用共置存在本质冲突——应用被调度下去后无法感知新包到达，只能靠集中式实体监控。
  - 机制：包到达时若 LC 任务正在该核运行，中断向量匹配 `UINV`，直接跳转用户态 handler（开销极小）；若 LC 未运行，向量不匹配，转为传统内核中断，由 SBB 注册的自定义内核 handler 唤醒 LC 任务并触发调度器。
  - **工程难点**：UINTR 硬件并非为设备中断设计——`PIR` 在首次处理后会被清零，需重新置位才能持续接收；而硬件未提供低开销手段（自发送 `SENDUIPI` 或陷入内核都会丧失 kernel-bypass 优势）。SBB 采用 Aeolia 的方案：初始化时通过系统调用把 `UPID` 映射进用户地址空间，使用户态可在每次处理完中断后重写 `PIR` 重新武装。
  - **避免中断风暴**：利用网卡 auto-mask（中断触发后自动屏蔽直至显式重新武装），仅在收包阶段解除屏蔽。

- **Timer UINTR → 实现去中心化请求抢占**
  - 每个核使用本地 LAPIC timer，配置为与 NIC 相同的 `UINV` 向量，从而定时中断直接跳入用户态 handler、完全绕过内核。
  - 流程：每处理一个请求前武装一次性 one-shot timer（仅需写 `APIC_LVTT` 与 `APIC_TMICT` 两个已映射到用户态的寄存器，约 50 周期）；若请求在时间片内完成则重置，否则触发用户态定时中断，保存上下文、重新入队、切换到下一个请求。
  - 相比 Linux `setitimer`（约 3 μs，多次用户/内核切换）大幅降低开销。

- **两类 UINTR 共存的难题与解法（论文首创）**
  - 难点：每核只有一套 UINTR 寄存器资源（`UIHANDLER`、`UINV`），所有 UINTR 跳同一 handler，无法直接区分来源；`stui/ctui` 只能全局开关，无法单独禁用某一类。
  - 解法基于两个观察：① 现代网卡（Intel E810、ConnectX-5）触发中断后自动屏蔽直至重新武装；② 两类中断的活跃期可以**不重叠**。
  - 具体做法：按 Linux 中断模型把包处理切分为 top half / bottom half，并细粒度分阶段——**步骤 1 收包（NIC UINTR 开启）→ 步骤 2 L2–L4 解析 → 步骤 3 应用层处理（timer UINTR 开启）→ 步骤 4 构造响应 → 步骤 5 发送**。NIC 中断仅在最后一批请求处理完后重新开启；timer 仅在应用层执行期间开启（请求时长差异主要来自应用层）。
  - 三重收益：消除高 RPC 速率下的中断风暴；避免无谓的 timer 中断；每个阶段只有一个可能的中断源，从而可识别与控制。

### 2.3 调度策略：两级负载均衡（任务窃取 + 流迁移）

- **问题诊断**：网卡 RSS 的 flow-to-core 映射同时造成 ① 临时不均衡（单流突发）与 ② 持久不均衡（某些核被分到过多流）。单纯任务窃取只对 ① 有效，对 ② 会产生持续、过量的缓存一致性流量。
- **仿真证据**（Fig.4）：在 95% 负载下，任务窃取的平均窃取次数高达 15.80（混合策略仅 1.23），被窃任务占比 90.90% vs 2.10%，性能远逊于 `JBSQ(2)`；被窃包集中在少数几条流上，印证 RSS 随机性导致的持久倾斜。
- **策略设计**：
  - **流迁移**（应对持久不均衡）：利用网卡 flow director，安装规则（约 10 μs/条，适合低频使用）把整条流重定向到空闲 worker 的 Rx 队列。
  - **增强型任务窃取**（应对临时不均衡）：包级、细粒度。
  - **victim 状态机**：`Light / Busy / Overloaded`，基于收包数与水位线判定，需**连续三轮**满足条件才迁移状态（抑制突发抖动）。`Busy` 表示临时过载，允许包级窃取；进入 `Overloaded` 后发布待处理流的标识，供他人迁移。
  - **stealer 行为**：Rx 队列空时扫描其他 worker；跳过 `Light`；对 `Busy` 窃取一半待处理请求；对 `Overloaded` 读取已发布的流信息并安装网卡规则迁移整条流。
- **任务窃取的五项优化**：
  - **O1 Owner-exclusive block**：队列前 16 项为属主专属、窃取者不可见，无需加锁；低负载（<16 包/轮）时完全不触发窃取，显著降低锁竞争与一致性开销。
  - **O2 避免多窃取**：窃取者拿到请求后直接处理，不再入本地队列，杜绝同一请求被反复窃取。
  - **O3 批量出队/窃取**：每次持锁处理/窃取一批（如 16 个），摊薄锁开销、提升局部性。
  - **O4 try-lock**：失败不自旋，立即尝试下一个候选。
  - **O5 局部性感知的 victim 选择**：从上轮成功窃取的 victim 开始扫描，各 worker 起始点不同以分散竞争。
- **基线参数**：请求队列容量 64（对齐 DPDK 最优批收包大小），FIFO 处理，队列加锁。

### 2.4 实现
- 内核补丁基于 Linux 6.12.20，**2,095 LOC**，主要增加 NIC 与 timer UINTR 支持；运行时基于 **DPDK v25.07.0**，**4,343 LOC**，含轻量 TCP/UDP 协议栈。
- 应用需改写，只需实现三个 hook：`init_application()`、`process_request()`、`build_response()`。
- 另实现完整客户端发包器（未修改内核 + DPDK），**2,067 LOC**，可配置请求时长分布、到达模式与发送速率。

## 3. 实验设计

- **测试床（RFC 2544 双机直连）**：
  - 服务器：2 个 NUMA 节点，各 64 核 Intel Xeon Platinum 8592 @1.9 GHz；关闭 TurboBoost、CPU idle states、频率调节、透明大页；
  - 客户端：单路 Intel Xeon 6780E，144 物理核 @3.0 GHz；
  - 两端均配 100 Gbps Intel E810 网卡；**Caladan 因在 Intel 网卡上性能退化，按其官方建议改用 ConnectX-5**；
  - 内核/系统：SBB 为 Ubuntu 24.04 + 自研内核；Shinjuku/Concord 用 Ubuntu 18.04 + kernel 4.4.0；Caladan/TQ 用 Ubuntu 22.04 + kernel 5.15.0；Skyloft 用 Ubuntu 24.04 + 自研内核。
- **工作负载**（Table 3）：
  - 合成：`Fixed(1)`（全部 1 μs，低离散度）、`High Bimodal`（仿 YCSB Workload A，50
