---
title: "{Söze}: One Network Telemetry Is All You Need for Per-flow Weighted Bandwidth Allocation at Scale"
title_zh: Söze：仅凭网络遥测即可实现大规模按流加权带宽分配
authors: "Weitao Wang, T. S. Eugene Ng"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-wang-weitao.pdf"
tags: ["query:dcn-comm"]
score: 7.0
evidence: 数据中心网络中的按流加权带宽分配
tldr: 加权带宽分配在现代数据中心网络中有广泛用途，但在大规模云环境中实现敏捷精确的分配极具挑战。Söze提出一种轻量去中心化系统，仅利用商用以太网交换机的简单网络遥测信息，无需逐流、拓扑或路由知识即可计算并执行加权带宽分配。仿真与测试床实验表明其能改善TPC-H作业完成情况，为大规模数据中心网络调度提供了简洁有效的机制。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-001.webp\", \"caption\": \"\", \"page\": 5, \"index\": 1, \"width\": 2068, \"height\": 1836}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-002.webp\", \"caption\": \"\", \"page\": 6, \"index\": 2, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-003.webp\", \"caption\": \"\", \"page\": 6, \"index\": 3, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-004.webp\", \"caption\": \"\", \"page\": 7, \"index\": 4, \"width\": 2651, \"height\": 917}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-005.webp\", \"caption\": \"\", \"page\": 8, \"index\": 5, \"width\": 1843, \"height\": 587}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-006.webp\", \"caption\": \"\", \"page\": 9, \"index\": 6, \"width\": 1494, \"height\": 818}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-007.webp\", \"caption\": \"\", \"page\": 9, \"index\": 7, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-008.webp\", \"caption\": \"\", \"page\": 9, \"index\": 8, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-009.webp\", \"caption\": \"\", \"page\": 10, \"index\": 9, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-010.webp\", \"caption\": \"\", \"page\": 10, \"index\": 10, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-011.webp\", \"caption\": \"\", \"page\": 10, \"index\": 11, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-012.webp\", \"caption\": \"\", \"page\": 10, \"index\": 12, \"width\": 1014, \"height\": 750}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-013.webp\", \"caption\": \"\", \"page\": 10, \"index\": 13, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-014.webp\", \"caption\": \"\", \"page\": 10, \"index\": 14, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-015.webp\", \"caption\": \"\", \"page\": 10, \"index\": 15, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-016.webp\", \"caption\": \"\", \"page\": 10, \"index\": 16, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-017.webp\", \"caption\": \"\", \"page\": 11, \"index\": 17, \"width\": 863, \"height\": 645}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-018.webp\", \"caption\": \"\", \"page\": 11, \"index\": 18, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-019.webp\", \"caption\": \"\", \"page\": 11, \"index\": 19, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-020.webp\", \"caption\": \"\", \"page\": 11, \"index\": 20, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-021.webp\", \"caption\": \"\", \"page\": 11, \"index\": 21, \"width\": 1093, \"height\": 744}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-022.webp\", \"caption\": \"\", \"page\": 11, \"index\": 22, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-023.webp\", \"caption\": \"\", \"page\": 12, \"index\": 23, \"width\": 2100, \"height\": 1050}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-024.webp\", \"caption\": \"\", \"page\": 12, \"index\": 24, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-025.webp\", \"caption\": \"\", \"page\": 12, \"index\": 25, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-026.webp\", \"caption\": \"\", \"page\": 12, \"index\": 26, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-027.webp\", \"caption\": \"\", \"page\": 12, \"index\": 27, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-028.webp\", \"caption\": \"\", \"page\": 12, \"index\": 28, \"width\": 2042, \"height\": 1458}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-029.webp\", \"caption\": \"\", \"page\": 12, \"index\": 29, \"width\": 2100, \"height\": 1500}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-030.webp\", \"caption\": \"\", \"page\": 12, \"index\": 30, \"width\": 1500, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-031.webp\", \"caption\": \"\", \"page\": 12, \"index\": 31, \"width\": 1500, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-032.webp\", \"caption\": \"\", \"page\": 13, \"index\": 32, \"width\": 1500, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-033.webp\", \"caption\": \"\", \"page\": 13, \"index\": 33, \"width\": 1500, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-034.webp\", \"caption\": \"\", \"page\": 14, \"index\": 34, \"width\": 1331, \"height\": 799}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-035.webp\", \"caption\": \"\", \"page\": 14, \"index\": 35, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-036.webp\", \"caption\": \"\", \"page\": 14, \"index\": 36, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-037.webp\", \"caption\": \"\", \"page\": 14, \"index\": 37, \"width\": 2100, \"height\": 1500}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-038.webp\", \"caption\": \"\", \"page\": 14, \"index\": 38, \"width\": 1650, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-039.webp\", \"caption\": \"\", \"page\": 14, \"index\": 39, \"width\": 1650, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-040.webp\", \"caption\": \"\", \"page\": 14, \"index\": 40, \"width\": 1650, \"height\": 1350}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-041.webp\", \"caption\": \"\", \"page\": 14, \"index\": 41, \"width\": 1350, \"height\": 1200}, {\"url\": \"assets/figures/osdi-2025-usenix/osdi-2025-s-ze-one-network-telemetry-is-all-you-need-for-per-flow-weighted-bandwidth-allocation-at-scale/fig-042.webp\", \"caption\": \"\", \"page\": 14, \"index\": 42, \"width\": 1350, \"height\": 1200}]"
motivation: 大规模云环境中实现敏捷精确的按流加权带宽分配十分困难，传统方法依赖逐流或拓扑信息。
method: Söze利用商用交换机简单遥测信息，在无需逐流、拓扑或路由知识下计算并执行加权带宽分配。
result: 通过仿真与测试床实验验证有效性，改善了TPC-H作业完成情况。
conclusion: 轻量去中心化的遥测驱动方法可在大规模数据中心网络中高效实现加权带宽分配。
---

## Abstract
Weighted bandwidth allocation is a powerful abstraction that has a wide range of use cases in modern data center networks. However, realizing highly agile and precise weighted bandwidth allocation for large-scale cloud environments is fundamentally challenging. In this paper, we propose Söze, a lightweight decentralized weighted bandwidth allocation system that leverages simple network telemetry features of commodity Ethernet switches. Given the flow weights, Söze can effectively use the telemetry information to compute and enforce the weighted bandwidth allocations without per-flow, topology, or routing knowledge. We demonstrate the effectiveness of Söze through simulations and testbed experiments, improving TPC-H jobs completion time by up to 0.59× and 0.79× on average.

---

## 论文详细总结（自动生成）

# Söze 论文结构化总结

## 1. 核心问题与整体含义

- **研究背景**：现代数据中心网络中的传输层通常追求固定目标，例如公平共享带宽，实际分配主要由流量模式决定。但云应用（分布式训练、Spark/Hadoop、数据库、RPC 等）的传输目标多样且动态变化，固定公平策略难以满足差异化需求。
- **核心问题**：加权带宽分配是一种强有力的抽象，可支持关键路径优先、尾延迟缓解、SLO 保障、最短流优先等场景；但在大规模云环境中实现**细粒度、高敏捷、高可扩展、高精度**的按流加权分配非常困难。
- **现有方案局限**：
  - 基于交换机包调度的 WFQ/WRR 等方案受限于交换机每流队列数量少、权重粒度粗、硬件资源有限，难以扩展到大量流。
  - 基于逻辑集中式带宽分配器加限速器的方案需要收集全局信息并求解优化，通信延迟和计算开销高，流到达、结束或权重变化时缺乏敏捷性。
- **论文整体含义**：Söze 提出一种轻量、去中心化的加权带宽分配系统，仅利用商用交换机已有的网络遥测（INT）功能，特别是队列延迟信号，在无需逐流状态、拓扑或路由知识的情况下，使各流独立收敛到加权 max-min 公平分配。其核心主张是：**沿流路径的一个网络遥测信号就足以协调多个发送端并实现瓶颈处的加权带宽分配**。

## 2. 方法论：核心思想与关键技术细节

### 2.1 核心思想

- 不把 INT 仅用于拥塞监测，而是将其**重新用作发送端之间的隐式信息交换通道**：链路队列延迟由所有竞争流共同决定，所有流观察到同一信号，从而可据此协调速率。
- 将加权分配目标拆解为两个可分布式验证的条件：
  - 链路到达率等于链路带宽，保证满利用率；
  - 所有流的“速率/权重”（rate-per-weight）相等，保证加权公平。
- 利用**瓶颈跳的队列延迟最高**这一性质，仅保留路径上的最大每跳队列延迟（maxQD），即可让流在瓶颈处执行加权分配，无需知道瓶颈具体在哪一跳。

### 2.2 单交换机场景的关键推导

- 目标：流 \(f_k\) 的速率满足  
  \[
  r_k=\frac{w_k}{\sum_i w_i}B
  \]
  其中 \(w_k\) 为权重，\(B\) 为链路带宽。
- 等价的分布式条件为：
  \[
  \sum_i r_i=B,\quad \frac{r_0}{w_0}=\frac{r_1}{w_1}=\cdots=\frac{r_n}{w_n}
  \]
- 队列延迟 \(D\) 的动态可表示为：
  \[
  \frac{dD}{dt}=\frac{R-B}{B},\quad \forall D>0
  \]
  其中 \(R\) 为链路到达率。只要队列延迟稳定在非零值，链路到达率即等于带宽。
- 将“速率/权重”映射为目标队列延迟：
  \[
  T\left(\frac{r}{w}\right)
  =p\cdot\frac{\ln(\alpha)-\ln(r/w)}{\ln(\alpha)-\ln(\beta)}+k
  \]
  该函数单调递减：rate-per-weight 越大，目标队列延迟越小。不同队列延迟对应不同加权公平份额。
- 速率更新函数：
  \[
  U\left(\frac{r}{w},D\right)
  =\left(\frac{T^{-1}(D)}{r/w}\right)^m
  \]
  每个流根据观测到的队列延迟与自身当前 rate-per-weight 计算乘性调整比例，更新发送速率。
- 算法流程（文字化）：
  1. 发送端接收带有 INT 信号的 ACK；
  2. 若距上次更新超过一个 RTT，则解析 maxQD；
  3. 根据目标函数反函数和当前 rate-per-weight 计算调整比例；
  4. 以乘性方式更新速率；
  5. 重复直至收敛。
- 收敛条件：
  - 加权公平收敛：\(0<m<2\)；
  - 目标队列延迟收敛：\(p>\frac{\Delta t}{2}[\ln(\alpha)-\ln(\beta)]\)。

### 2.3 任意网络场景：maxQD 与瓶颈识别

- 加权 max-min 公平定义：对任意流，若增加其速率会导致某条满足 \(r_{f'}/w_{f'}\le r_f/w_f\) 的流速率下降，则当前分配为加权 max-min 公平。
- 瓶颈跳性质：达到加权 max-min 公平时，每条流在其瓶颈跳上具有最大的 rate-per-weight，而在非瓶颈饱和跳上不具有最大 rate-per-weight。
- 由于目标函数单调递减，瓶颈跳的队列延迟高于非瓶颈跳。因此路径上的**最大每跳队列延迟 maxQD** 必然来自瓶颈跳。
- 交换机只需在数据包头部比较并保留 maxQD，接收端将其放入 ACK 反射回发送端；发送端使用 maxQD 执行统一速率更新算法。该过程不需要知道瓶颈跳位置，也不需要拓扑或路由信息。
- 定理：在任意网络中，若 \(0<m<2\) 且 \(p>\frac{\Delta t}{2}[\ln(\alpha)-\ln(\beta)]\)，Söze 收敛到加权 max-min 公平分配。

### 2.4 系统实现要点

- **交换机侧**：在 Tofino 交换机上仅用 9 行代码实现队列延迟 INT 信号，使用低通滤波采样，默认输出 10 μs 平滑队列延迟。
- **主机侧**：
  - Linux 内核模块原型 241 行，替换默认 Cubic，通过 TCP 选项传递权重；
  - eRPC 用户态实现 1972 行，替换 Timely，增加头部字段和队列信号处理；
  - NS-3 仿真器实现 maxQD 逐跳更新与 ACK 反射。
- **参数设置**：典型 \(p=20\mu s\)、\(k=3\mu s\)、\(m=0.25\)；交换机端口 25 Gbps。参数选择原则：\(p\) 控制信号粒度与振荡，\(k\) 保证满利用率，\(m\) 权衡收敛速度与稳定性。
- **更新粒度**：采用每包更新而非每 RTT 更新，以加快权重执行和收敛。

## 3. 实验设计

### 3.1 实验平台与场景

- **eRPC 测试床**：
  - 4 台带 DPDK 能力 CX-4 NIC 的服务器连接 1 台 Tofino-1 可编程交换机，星型拓扑；
  - 流为 RPC write 请求，请求 7 MB、响应 32 字节；
  - 多发送端向同一目的端发送时，交换机出口成为瓶颈；
  - 端口带宽 25 Gbps。
- **NS-3 大规模仿真**：
  - 1024 台服务器、320 台交换机，fat-tree 拓扑；
  - 100 Gbps 链路、1 μs 链路延迟、32 MB 缓冲、1000 字节包负载；
  - 用于 TPC-H、FCT slowdown、微基准和大规模可扩展性实验。
- **应用场景**：
  - 作业内关键路径优先；
  - coflow 内 straggler 识别与缓解；
  - 多作业共享同一瓶颈时的资源再分配；
  - 多作业间的利他调度；
  - 最短流优先；
  - TPC-H 22 个作业的 DAG 加速。
- **拥塞控制与微基准场景**：
  - step-in/step-out 动态流加入与退出；
  - Google RPC 工作负载下 40% 和 80% 负载的 FCT slowdown；
  - 加权 max-min 公平性、瓶颈跳变化；
  - 权重粒度、非整数权重、多连接对比；
  - 敏捷性收敛时间；
  - 可扩展性：拓扑规模 K=4 至 K=16，流数 10 至 100 万。

### 3.2 Benchmark 与对比方法

- **对比协议/系统**：
  - Timely：eRPC 默认拥塞控制；
  - DCQCN：RDMA 拥塞控制；
  - HPCC：基于 INT 的高精度拥塞控制；
  - WRR + AIMD：交换机加权轮询调度结合 AIMD 速率控制；
  - 集中式 water-filling 算法：作为可扩展性对比；
  - 多连接方案（Multi-Conn）：用整数连接数近似权重；
  - 交换机物理队列方案：4 个物理队列打包流并设权重。
- **评价指标**：
  - 作业完成时间 JCT；
  - 流完成时间 FCT slowdown；
  - 链路利用率；
  - RTT；
  - 收敛时间；
  - 加权公平性/速率准确性；
  - 权重粒度；
  - 可扩展性求解/收敛时间。

## 4. 资源与算力

- 论文**未提及 GPU 型号、数量或训练时长**；该工作属于数据中心网络传输与带宽分配系统研究，不涉及模型训练算力。
- 文中报告了网络实验资源：
  - eRPC 测试床：4 台服务器、CX-4 NIC、1 台 Tofino-1 交换机、25 Gbps 端口；
  - NS-3 仿真：1024 服务器、320 交换机、100 Gbps 链路。
- 论文未明确报告服务器 CPU 配置、内存、实验运行总时长或能耗等细节。

## 5. 实验数量与充分性

- **实验数量**：主要实验至少包括：
  - 6 类应用案例：关键路径、straggler、共享瓶颈、利他调度、最短流优先、TPC-H；
  - 3
