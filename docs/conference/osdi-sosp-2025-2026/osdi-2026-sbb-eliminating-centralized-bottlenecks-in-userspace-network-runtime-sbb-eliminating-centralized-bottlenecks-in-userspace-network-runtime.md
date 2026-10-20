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
score: 4.0
evidence: 去中心化用户态网络运行时，面向高吞吐与低延迟
tldr: 针对用户态网络运行时依赖集中式组件、随着CPU工作线程增多而成为扩展瓶颈的问题，SBB提出一种完全去中心化的运行时，在请求抢占、CPU分配和跨核负载均衡三类调度上重新设计系统机制与调度策略。实验表明，它能够同时实现高吞吐、低延迟、高CPU效率和高可扩展性，为大规模数据中心网络通信运行时提供了可扩展的基础。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 现有用户态网络运行时依赖集中式组件，随着CPU工作线程增多会成为扩展瓶颈，限制系统性能。
method: SBB提出完全去中心化的运行时，在请求抢占、CPU分配和跨核负载均衡三类调度上重新设计系统机制与调度策略。
result: 实验表明，SBB能够同时实现高吞吐、低延迟、高CPU效率和高可扩展性。
conclusion: 该工作为大规模数据中心网络通信运行时提供了可扩展的去中心化设计思路。
---

## Abstract
To achieve high throughput, low latency, and high CPU efficiency, userspace network runtimes must perform three types of scheduling: 1) request preemption to minimize tail latency, 2) CPU allocation among services to avoid wasting CPU cycles when the load is low, and 3) request load balancing across worker cores to achieve work conservation. However, prior designs rely on centralized components, which inevitably become scalability bottlenecks as the number of CPU workers increases, limiting system performance scaling. This paper presents SBB, a purely decentralized userspace network runtime that simultaneously delivers high performance, high CPU efficiency, and high scalability by advancing in both system mechanism and scheduling policy. For system mechanism, SBB leverages the emerging User Interrupt mechanism in a novel way, delivering two types of device interrupts to the userspace runtime: 1) user-level timer interrupts for request preemption, and 2) user-level NIC interrupts for packet arrival to perform CPU allocation. For scheduling policy, SBB introduces a two-level algorithm, which marries flow migration (for persistent imbalance) with task stealing (for temporary imbalance), challenging the conventional wisdom that centralized load balancing outperforms decentralized approaches. Evaluation shows that SBB achieves 1.7× to 5.2× higher throughput when scaling to 48 cores, while meeting the same tail latency target.

---

## 论文详细总结（自动生成）

# SBB：消除用户态网络运行时中的集中式瓶颈 —— 论文中文总结

## 1. 核心问题与研究背景

- **背景**：现代数据中心网卡带宽已达数百 Gbps，而单核乃至十余核 CPU 难以匹配，网络运行时必须具备良好的多核可扩展性；同时尾延迟进入微秒级，要求细粒度调度。
- **三类必要调度**：
  1. **请求抢占（request preemption）**：防止长请求造成短请求的队头阻塞，降低尾延迟；
  2. **CPU 分配（CPU allocation）**：LC（延迟敏感）与 BE（尽力而为）应用共置时，低负载下把空闲核让给 BE，突发时迅速收回；
  3. **请求负载均衡（request load balancing）**：跨 worker 核实现 work conservation。
- **核心问题**：已有系统为完成上述三类调度，普遍依赖**独立集中式实体**——集中式定时器（Shinjuku 等）、集中式 monitor（Caladan 的 iokernel）、集中式 dispatcher（Shinjuku/Concord 等）。随着 worker 核数增加，这些实体必然成为扩展性瓶颈。
- **实测证据（§2.3.2，图 3）**：
  - 集中式定时器：worker 超过 16 个后吞吐停止增长甚至下降，5 μs 抢占间隔内无法向所有核发 IPI；
  - 集中式 monitor：on-off 突发负载下 Caladan 无法及时重分配核，SLO 违例；
  - 集中式 dispatcher：单包处理约 200 ns，系统吞吐被硬性封顶在约 5 MRPS；
  - **多 dispatcher 无济于事**：分组后吞吐反而随组数增加而下降（负载不均重现）；
  - **User IPI 不可扩展**：即使增加发送核，中断发送速率仍受单一 NUMA 约束而有上限。
- **已有工作的三点不足**：① 未验证的假设（认为多核可扩展集中式实体）；② 缺乏微秒级、去中心化的调度事件通知机制；③ 缺乏有效的去中心化调度策略（集中式负载均衡更接近理论最优 FCFS 共享队列，但忽略可扩展性约束）。
- **整体含义**：SBB 提出一个**完全去中心化**的用户态网络运行时，同时实现高吞吐、低延迟、高 CPU 效率与高可扩展性，在机制与策略两个层面同时推进。

## 2. 方法论

### 2.1 核心思想

- 用新兴的 **User Interrupt（UINTR，Intel Sapphire Rapids 引入）** 把两类**设备中断**直接投递到用户态：
  - **NIC 用户态中断** → 替代轮询，实现及时的 CPU 分配；
  - **每核 LAPIC 定时器用户态中断** → 实现去中心化的请求抢占。
- 负载均衡采用**两级策略**：任务窃取（临时性不均）+ 流迁移（持久性不均），挑战“集中式负载均衡必然优于去中心化”的传统认知。

### 2.2 关键技术细节

**（1）UINTR 基础机制（§2.2）**
- 每核 6 个 MSR：`UIRR`（待处理中断位图）、`UIHANDLER`（用户态处理函数地址）、`UISA`（栈调整）、`UITTADDR`（UITT 地址）、`UIMISC`（UITTSZ/UINV）、`UPIDADDR`（UPID 地址）。
- 每线程有 UPID（存 PIR 待处理位与 NV 向量）与 UITT（表项含 UV 与 ADDRUPID）；发送用 `SENDUIPI` 指令。
- 投递三阶段：**识别**（向量是否等于 UINV）→ **通知处理**（PIR 拷入 UIRR 并清空 PIR）→ **投递**（返回用户态时跳转到 UIHANDLER）。

**（2）NIC 用户态中断与 CPU 分配（§4.2）**
- 动机：轮询与多应用共置存在根本冲突——被调度下核的应用无法感知新包到达。
- 两个必要条件：NIC 中断向量需匹配 `UINV`（初始化时读取各队列向量并设置）；`PIR` 必须非零——但 PIR 在首次处理周期即被清空。
- **重新武装（re-arm）方案**：采用 Aeolia 的做法，初始化时通过系统调用把 UPID 映射进用户地址空间，每次处理完 NIC 用户中断后由用户程序重写 PIR，避免 `SENDUIPI` 自投或陷入内核。
- **LC 任务未运行时的路径**：向量不匹配 → 走传统内核中断 → SBB 注册的自定义内核 handler 唤醒 LC 任务并调用调度器，从而把核抢回。
- **避免中断风暴**：利用网卡 auto-mask（类似中断抑制），仅在收包步骤 1 解除屏蔽，其余阶段保持屏蔽。

**（3）LAPIC 定时器用户态中断与请求抢占（§4.3）**
- 传统 `setitimer` 路径需多次用户/内核切换，开销约 3 μs；SBB 让每个 worker 把 LAPIC 定时器配置为与 NIC 中断相同的向量（即 `UINV`），从而直接跳到用户态 handler，完全绕过内核。
- 流程：处理每个请求前武装一次性定时器；若在时间片内完成则重置定时器；否则触发用户态定时器中断 → 保存上下文入本地队列 → 调度下一个请求。
- 编程开销仅 **50 cycles**（只需写 `APIC_LVTT` 与 `APIC_TMICT` 两个已映射到用户态的寄存器）。

**（4）两类 UINTR 共存（§4.4，SBB 首次尝试）**
- 难点：单核只有一套 UINTR 寄存器，所有 UINTR 跳到同一 handler，无法直接区分来源；`stui`/`ctui` 只能全局开关。
- 两个观察：① 现代网卡触发中断后自动屏蔽，直到显式重新武装；② 两类中断的活跃期可不重叠。
- 方案：把包处理划分为 **top half / bottom half**——NIC 中断 handler 只置 pending 标志即返回；出队、协议栈解析、应用层处理、回包由 worker 线程执行；仅在收包步骤启用 NIC 中断，仅在应用层执行期启用定时器中断。由此每个阶段只有一种可能的中断源，可被直接识别与控制。
- 附带收益：高 RPC 速率下无中断风暴、避免不必要的定时器中断。

**（5）两级负载均衡策略（§4.5）**
- **问题分解**：RSS 的流到核映射同时造成 ① 临时不均（单流突发）与 ② 持久不均（某些核分到过多流）；任务窃取只适合前者，对后者会产生过量缓存一致性流量。
- **流迁移**：利用商用网卡的 flow director，安装 5 元组规则把整条流重定向到指定 Rx 队列，规则安装约 **10 μs**，适合低频使用。
- **worker 状态机**：`Light` / `Busy` / `Overloaded`，需连续 3 轮满足水位条件才迁移状态（抑制突发抖动）；进入 Overloaded 后发布下一条待处理流的标识供他人迁移。
- **窃取者行为**：发现自身 Rx 队列空即扫描其他 worker；`Light` 跳过；`Busy` 则窃取队列中一半待处理请求（包级）；`Overloaded` 则读取已发布的流信息并安装 NIC 规则做流级迁移。
- **任务窃取优化 O1–O5**：
  - O1 所有者专属块：每队列前 16 项对窃取者不可见、免锁，低负载时避免无谓窃取；
  - O2 避免同一任务被多次窃取：窃取者直接处理，不再入自己队列；
  - O3 批量出队/批量窃取（如一次 16 个、窃取一半）以摊薄锁开销；
  - O4 使用 try-lock，失败即跳到下一个候选，不自旋；
  - O5 局部性感知的受害者选择：从上轮成功窃取的受害者开始扫描，并让各 worker 起点错开以分散竞争。

**（6）实现（§5）**
- 内核补丁基于 **Linux 6.12.20，2,095 LOC**（新增 NIC 与定时器 UINTR 支持）；运行时基于 **DPDK v25.07.0，4,343 LOC**，含轻量 TCP/UDP 协议栈。
- 应用需重写并实现三个 hook：`init_application()`、`process_request()`、`build_response()`。
- 另实现功能完整的客户端发包器（2,067 LOC，基于未修改内核 + DPDK）。

## 3. 实验设计

- **测试床**：遵循 RFC 2544，两台直连机器。服务器：双 NUMA 节点、每节点 64 核 Intel Xeon Platinum 8592 @1.9 GHz（关闭 TurboBoost、idle 状态、调频、透明大页）；客户端：单路 Intel Xeon 6780E，144 物理核 @3.0 GHz；两端均为 **100 Gbps Intel E810** 网卡（Caladan 因在 Intel 网卡上性能退化，改用 ConnectX-5 以取得最佳性能）。
- **系统/基线**（按调度类型分别对比，负载均衡在所有实验中均评估）：
  - 请求抢占：**Shinjuku、Concord、TQ**；
  - CPU 分配：**Skyloft、Caladan、Caladan-DL**（Delay Range 策略）；
  - 负载均衡：模拟器中的 **Single queue（理论上界）、JBSQ(2)（最佳集中式）、Task stealing（最佳先前去中心化）、Hybrid（SBB）**。
- **工作负载**（表 3）：
  - 合成：`Fixed(1)`（1 μs 固定，低离散）、`High Bimodal`（仿 YCSB Workload A，短 1 μs 占 50%/长 100 μs 占 50%）、`Extreme Bimodal`（仿 Facebook USR，短 0.5 μs 占 99.5%/长 500 μs 占 0.5%）；
  - 真实应用：**RocksDB**（GET 1.25 μs 的轻尾与重尾两种配置，SCAN 613 μs 占 50%）；
  - 共置场景：**Memcached（LC）+ PARSEC swaptions（BE）**，Poisson 到达；另测多 LC 任务共置。
- **指标与 SLO**：固定 p99.9  slowdown 目标（50×）或 p99.9 时延目标（50/100 μs）下的最大吞吐；默认 16 线程独占核，扩展性实验用 32/48 核。
- **微基准**：往返时延分解（polling / UINTR / 传统中断）、抢占机制开销对比（表 4）、负载均衡策略消融（图 13）。

## 4. 资源与算力

- 本文**不涉及 GPU 训练**，未报告任何 GPU 型号、数量或训练时长。
- 明确给出的硬件资源为 CPU 测试床：服务器双路 64 核 Xeon Platinum 8592（共 128 核，2 NUMA 节点
