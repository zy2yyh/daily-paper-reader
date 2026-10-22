---
title: "Chop Chop: Byzantine Atomic Broadcast to the Network Limit"
title_zh: Chop Chop：达到网络极限的拜占庭原子广播
authors: "Martina Camaioni, Rachid Guerraoui, Matteo Monti, Pierre-Louis Roman, Manuel Vidigueira, Gauthier Voron"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi24-camaioni.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 达到网络极限的拜占庭原子广播通信原语
tldr: 针对状态机复制中原子广播需同时完成排序、认证和去重而难以达到网络极限的问题，Chop Chop提出带认证内存池的拜占庭原子广播系统。它通过蒸馏批处理摊薄排序、认证与去重开销，即使处理8字节小消息也能接近线速。该工作展示了分布式通信原语的高效实现，但其面向拜占庭容错复制而非数据中心集合通信或拓扑感知路由，对集合通信调度仅有间接参考价值。
source: OSDI-2024-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-chop-chop-byzantine-atomic-broadcast-to-the-network-limit/fig-001.webp\", \"caption\": \"\", \"page\": 1, \"index\": 1, \"width\": 875, \"height\": 356}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-chop-chop-byzantine-atomic-broadcast-to-the-network-limit/fig-002.webp\", \"caption\": \"\", \"page\": 1, \"index\": 2, \"width\": 875, \"height\": 360}]"
motivation: 状态机复制中的原子广播需对消息排序、认证和去重，现有协议难以在保持拜占庭容错时达到网络线速。
method: Chop Chop采用带认证的内存池和新式蒸馏批处理，摊薄消息排序、认证与去重的开销。
result: 系统即使处理8字节小消息也能接近线速，性能接近不提供排序和拜占庭容错的通信协议。
conclusion: 该工作为分布式系统提供高效拜占庭原子广播原语，但与数据中心集合通信主题仅部分相关。
---

## Abstract
At the heart of state machine replication, the celebrated technique enabling decentralized and secure universal computation, lies Atomic Broadcast, a fundamental communication primitive that orders, authenticates, and deduplicates messages. This paper presents Chop Chop, a Byzantine Atomic Broadcast system that uses a novel authenticated memory pool to amortize the cost of ordering, authenticating and deduplicating messages, achieving "line rate" (i.e., closely matching the complexity of a protocol that does not ensure any ordering, authentication or Byzantine resilience) even when processing messages as small as 8 bytes. Chop Chop attains this performance by means of a new form of batching we call distillation. A distilled batch is a set of messages that are fast to authenticate, deduplicate, and order. Batches are distilled using a novel interactive protocol involving brokers, an untrusted layer of facilitating processes between clients and servers. In a geo-distributed deployment of 64 medium-sized servers, Chop Chop processes 43,600,000 messages per second with an average latency of 3.6 seconds. Under the same conditions, state-of-the-art alternatives offer two orders of magnitude less throughput for the same latency. We showcase three simple Chop Chop applications: a Payment system, an Auction house and a "Pixel war" game, respectively achieving 32, 2.3 and 35 million operations per second.

---

## 论文详细总结（自动生成）

# Chop Chop：达到网络极限的拜占庭原子广播 —— 论文深度总结

## 1. 核心问题与整体含义（研究动机与背景）

- **背景定位**：状态机复制（SMR）是实现去中心化、安全、通用计算的核心技术，而原子广播（Atomic Broadcast）是 SMR 的基础通信原语，负责对消息进行**排序（ordering）、认证（authentication）和去重（deduplication）**。
- **现实落差**：全球化部署下，经典实现（BFT-SMaRt、HotStuff）每秒仅能处理数千条消息，比互联网大型集中式服务（每秒数百万请求）低约三个数量级（图 1）。即便引入 Narwhal 等内存池（mempool）批处理方案，仍与"线速（line rate）"相差约 2.5 个数量级。
- **瓶颈识别**：论文指出批处理只摊薄了**排序**成本，但**认证与去重**成本并未摊薄——每条消息仍携带独立公钥、签名和序列号。例如支付系统中，12 字节的有效载荷因公钥、签名、序列号膨胀到 140 字节，**91% 的带宽浪费在完整性与去重上**。
- **核心问题**：原子广播与无序、无认证、无拜占庭容错的分发之间的巨大差距，是否是原子性的固有代价？论文给出否定答案。

## 2. 方法论：核心思想与关键技术细节

### 2.1 核心思想：蒸馏批处理（Distillation）

- 提出**蒸馏批（distilled batch）**：将批内所有独立签名聚合为**一个聚合签名（aggregate signature）**，所有序列号聚合为**一个聚合序列号（aggregate sequence number）**。
- 利用 BLS 多签名方案 [10]：同一消息上的多个签名可非交互聚合，聚合公钥与聚合签名可**常数时间验证**，从而把认证成本从"每条消息"降为"每批一次"。

### 2.2 蒸馏流程（无故障简化版）

1. 客户端 χᵢ 选择序列号 kᵢ（大于其此前用过的所有序列号），将 (kᵢ, mᵢ) 发送给 broker β。
2. β 计算聚合序列号 **k = maxᵢ kᵢ**，构造批提案 **B = [(x₁,k,m₁), …, (x_b,k,m_b)]**，其中 xᵢ 是客户端在公钥目录中的数值标识。
3. β 将 B 回传给所有客户端；每个 χᵢ 对 **H(B)** 生成多签名 sᵢ 并回传。
4. β 聚合得到 **s = ∏ᵢ sᵢ**，形成蒸馏批 **B̃ = [s, k, ((x₁,m₁), …, (x_b,m_b))]**。
5. 服务器收到 B̃ 后：插入 k 重建 B → 计算 H(B) → 用 xᵢ 查目录得公钥 → 聚合公钥 p = ∏ᵢ pᵢ → 用 (s, p, H(B)) 常数时间验证。

### 2.3 拜占庭容错增强（§4.2）

- **防伪造消息**：客户端签名前必须验证自己的消息正确出现在 B 中（借助 Merkle 树包含证明）。
- **Merkle 树优化**：β 只发送 Merkle 根 r、聚合序列号 k 及每个客户端的包含证明 pᵢ（O(log l) 大小），无需广播整个批；服务器拒绝同一客户端出现多条消息的批。
- **防客户端不签名（超时回退）**：客户端首次提交时附带不可聚合的独立签名 tᵢ；超时未签 r 的客户端消息以 (kᵢ, tᵢ) 形式保留，实现**部分蒸馏**，最坏退化为经典批处理。
- **防重放**：客户端同一时刻只广播一条消息；服务器记录每个客户端的最后序列号 k̄χ 与最后消息 m̄χ，仅当 k > k̄χ 且 m ≠ m̄χ 时才投递。
- **防序列号耗尽（合法性证明）**：定义"合法序列号"为小于服务器已投递批数的值；服务器对第 n 批投递签名，f+1 个签名组成证书 lₙ，任何进程可证明小于 n 的序列号合法。
- **Broker 崩溃**：客户端可切换到其他 broker（假设至少一个 broker 正确）。

### 2.4 提交阶段（§4.3）

- **见证（witness）**：broker 请求 f+1 个服务器对 B̃ 签名"见证分片"（同时声明批格式正确且可检索）；聚合为 witness 后连同批哈希提交给服务器端原子广播实例。非见证服务器可信任 witness，节省可信 CPU。
- **投递与响应**：服务器从原子广播获得哈希后取回 B̃ 并投递；随后签名投递证书，broker 收集 f+1 个签名后分发给客户端，客户端据此安全广播下一条消息。

### 2.5 工程优化（§5）

- **可靠 UDP**：broker 需维持 60 万级并发连接，TCP 不可行；自研 ACK 重传协议应对丢包。
- **EdDSA 批量验证**、**二叉树上搜索非法多签名**（对数时间定位）、**合法性证明缓存**。
- **标识符排序批处理**：按客户端 ID 排序，服务器线性时间验证 ID 严格递增，并支持并行去重。
- **批垃圾回收**：所有服务器投递后回收，受限于原子广播的无限内存模型约束。

## 3. 实验设计

### 3.1 场景与负载

- **跨云地理分布式部署**：64 台 AWS c6i.8xlarge 服务器，分布于 14 个 AWS 区域（开普敦、圣保罗、巴林、加拿大、法兰克福、弗吉尼亚北部、加州北部、斯德哥尔摩、俄亥俄、米兰、俄勒冈、爱尔兰、伦敦、巴黎）；负载 broker 部署在 OVH（另一云厂商），以更真实模拟互联网流量。
- **客户端模拟**：最多模拟 **2.57 亿客户端**；客户端运行在 t3.small 小机器上以获取真实端到端延迟。
- **工作负载**：默认 8 字节消息、65,536 条/批（约 736 KB），共 12 套实验环境，每套需安装 13 TB 合成负载。
- **三个应用用例**：支付系统（Payment）、拍卖行（Auction）、"像素战争"游戏（Pixel war）。

### 3.2 基线对比

| 基线 | 说明 |
|---|---|
| HotStuff [80] | 高吞吐原子广播（C++） |
| BFT-SMaRt [9] | 低延迟原子广播，类 PBFT（Java） |
| Narwhal-Bullshark | 最先进 DAG 内存池 + Bullshark（Rust），认证交由应用 |
| Narwhal-Bullshark-sig | 修改版 Narwhal，服务器端进行 Ed25519 批量签名验证，与 Chop Chop 保证对齐 |

- Chop Chop 分别部署在 HotStuff 与 BFT-SMaRt 之上，以隔离底层协议影响。

### 3.3 实验问题（6 个 RQ）

- RQ1 负载处理（吞吐-延迟曲线）；RQ2 蒸馏收益（消融 + 消息大小 8B–512B + 线速效率）；RQ3 服务器规模（8/16/32/64）；RQ4 整体资源效率（匹配总硬件资源）；RQ5 服务器故障下表现；RQ6 应用用例吞吐。

## 4. 资源与算力

- **无 GPU 参与**：这是纯 CPU 的分布式系统实验，全文未提及 GPU 型号或数量。
- **计算资源**：
  - 64 台 AWS **c6i.8xlarge** 服务器（Intel Xeon Platinum 8375C，32 vCPU / 16 物理核，64 GB 内存，12.5 Gb/s 带宽）。
  - OVH 侧负载 broker 机器（Linux kernel 5.4.0）。
  - 最多同时调度 **320 台机器**，横跨 2 个云厂商、25 个区域。
- **成本**：AWS 信用额度 25,000 美元；使用 OVH 节省超过 70,000 美元出口带宽费（每个数据点约 1,700 美元）。
- **训练时长**：无训练过程；每个数据点为 **5 次 × 2 分钟**运行的平均值（剔除预热/冷却）。
- **辅助工具**：自研 **silk** 点对点文件传输工具，将 13 TB 文件部署从 scp 的 68 小时缩短到约 30 分钟。

## 5. 实验数量与充分性

- **实验组数**：覆盖 6 个 RQ、4 个基线、4 种系统规模、5 种消息大小、3 个应用、2 类故障场景（服务器崩溃 1 台 / 1/3；客户端不参与蒸馏），加上资源匹配实验，总计约 **10 组以上不同维度的实验**。
- **充分性**：
  - 优点：跨云、跨大洲的真实地理分布式部署规模在学术论文中非常罕见；基线选择兼顾经典协议与最先进内存池；消融实验明确分离蒸馏贡献（1.5M → 44M op/s，29 倍提升）。
  - 数据可靠性：每点 5 次运行并报告标准差，复现了 Narwhal-Bullshark 原始实验结果以验证测试床可信度。
- **公平性**：
  - 论文明确讨论"匹配可信资源"与"匹配总资源"两种比较口径，分别用于压力测试（§6.3–6.5）与效率评估（§6.6），较为透明。
  - 潜在偏差：Chop Chop 使用不可信 broker 资源，与 Narwhal 的 trusted worker 在信任模型上不同，直接比较存在口径差异（论文承认缺乏明确的信任/非信任资源换算标准）。
  - 8 字节消息的极端场景占主导，对更大消息或混合负载的覆盖相对有限。

## 6. 主要结论与发现

- **吞吐突破**：Chop Chop 达到 **43,600,000 消息/秒**，平均延迟 3.6 秒；同等延迟下最先进方案吞吐低两个数量级。
- **接近线速**：在 4000 万 op/s 拐点之前，网络开销 **小于 8%**；全蒸馏把认证+序列化带宽成本降低 9.7 倍、CPU 成本降低 28.2 倍（理论估算）。
- **蒸馏是核心贡献**：无蒸馏时服务器瓶颈在 1.5M op/s，完全蒸馏后达 44M op/s（29 倍）；相比 Narwhal-Bullshark-sig（382k op/s）高约 3.9 倍（未蒸馏时）至两个数量级（蒸馏后）。
- **消息大小影响**：8B 时 44.3M op/s，32B 时 17.6M，128B 时 3.5M，512B 时 890k op/s，随消息增大近似线性下降。
- **故障容忍**：1 台服务器崩溃吞吐从 44M 降至 43M；1/3 崩溃降至 15M（降 66%）；全部客户端不参与蒸馏时退化至 1.5M op/s。
- **资源效率**：匹配 128 台总机器时，Chop Chop 达 4.6M op/s（服务器 CPU 仅约 5%），Narwhal-Bullshark-sig 为 679k op/s。
- **应用表现**：支付 32M op/s、像素战争 35M op/s、单线程拍卖

应用受限于其串行竞价/结算逻辑，吞吐明显低于支付与“像素战争”类可并行负载，但仍显著高于传统原子广播基线，说明蒸馏收益可迁移到真实业务语义，而非仅适用于纯吞吐微基准。

## 7. 贡献定位与批判性评价

### 7.1 主要贡献

- **问题重述贡献**：论文没有把“原子广播慢”简单归因于共识轮次或批处理不足，而是将瓶颈精确定位到**逐消息认证与去重**，并用带宽构成分析（12 字节载荷膨胀至 140 字节、91% 开销用于完整性与去重）支撑这一判断，问题定义清晰且具有工程可操作性。
- **协议贡献**：蒸馏批将 BLS 多签名与聚合序列号结合，使认证成本从 O(b) 降为 O(1)，同时通过 Merkle 包含证明、超时回退、部分蒸馏、合法性证书等机制维持拜占庭容错，形成了一个较完整的“可退化”设计：正常路径接近线速，异常路径退回经典批处理而不破坏安全性。
- **系统贡献**：可靠 UDP、EdDSA 批量验证、非法多签名对数定位、按 ID 排序的线性验证、批垃圾回收等工程优化，使协议在 60 万级并发连接和跨云部署下可运行；这些细节对后续高吞吐 BFT 系统具有直接复用价值。
- **实验贡献**：跨 14 个 AWS 区域、最多 2.57 亿模拟客户端、匹配总硬件资源的对比，提供了少见的地理分布式规模证据；同时复现 Narwhal-Bullshark 以校准测试床，增强了结果可信度。

### 7.2 局限与需要谨慎解读之处

- **信任模型差异**：Chop Chop 将负载 broker 置于不可信侧，而 Narwhal 的 worker 通常被视为可信组件；论文虽以“匹配总资源”和“匹配可信资源”两种口径讨论，但缺少公认的信任/非信任资源换算标准，因此跨系统绝对倍数的解读应保留余地。
- **客户端参与成本**：蒸馏要求客户端在线接收批、验证包含证明并生成多签名；在移动端、弱连接或大规模离线场景中，超时回退可能频繁触发，实际收益取决于客户端在线率与网络稳定性。论文的客户端模拟虽规模大，但主要运行于受控云环境。
- **故障模型覆盖**：实验覆盖服务器崩溃与客户端不参与蒸馏，但对拜占庭服务器、broker 作恶、网络分区、长尾延迟和对抗性调度等场景的覆盖相对有限；BLS 聚合签名的安全性也依赖公钥目录与可信设置假设。
- **负载代表性**：8 字节消息是突出认证开销的合理压力测试，但真实系统常包含变长消息、混合读写、热点账户与状态依赖；论文对 8B–512B 的扫描有帮助，仍不足以完全代表通用 SMR 工作负载。
- **工程复杂度与运维成本**：可靠 UDP、Merkle 证明、见证分片、合法性证书、批 GC 等机制增加了实现与调试复杂度；跨云部署和 13 TB 合成负载也意味着复现门槛较高。
- **比较公平性**：Chop Chop 分别叠加在 HotStuff 与 BFT-SMaRt 上，隔离了底层协议影响，但不同语言实现、批大小调优和网络栈差异仍可能影响横向比较；论文报告 5 次运行均值与标准差，已较规范，但未完全消除实现偏差。

## 8. 对后续研究的启示

- **“摊销认证”应成为 BFT 设计的首要优化目标之一**：在广域网上，签名与序列号开销往往比共识消息轮次更接近实际上限；将认证单位从“消息”提升到“批”，比单纯增大批大小更具结构性收益。
- **可退化设计比单点最优更重要**：蒸馏的正常路径依赖客户端配合，论文通过超时回退和部分蒸馏保证最坏情况仍不劣于经典批处理；这种“快速路径—安全回退”模式值得在其他共识协议中借鉴。
- **信任边界需要显式建模**：broker、见证服务器、客户端各自承担不同信任假设；未来工作可建立更细粒度的资源—信任换算框架，使不同系统间的效率比较更具可比性。
- **应用语义与批处理协同**：支付、拍卖、游戏等用例表明，原子广播的吞吐不仅取决于协议，也取决于业务是否允许并行去重与批内排序；将应用层语义纳入批构造，可能进一步释放蒸馏收益。
- **工程可复现性**：可靠 UDP、批量验证和非法签名定位等组件可沉淀为通用基础设施；若开源，将降低后续高吞吐 BFT 系统的实现门槛。

## 9. 总体结论

Chop Chop 的核心论点可以概括为：**原子广播与线速分发之间的差距，并非原子性的固有代价，而是逐消息认证与去重的实现选择造成的可消除开销**。论文通过蒸馏批将批内独立签名与序列号聚合为常数大小的认证材料，并在拜占庭容错、客户端不配合、服务器故障和 broker 崩溃等条件下给出回退机制，使系统在正常路径上达到数千万消息/秒、接近网络线速，在异常路径上退化为经典批处理而不牺牲安全性。

其实验规模——跨多云、跨大洲、数亿模拟客户端、与最先进 DAG 内存池方案及经典 BFT 协议的系统对比——为“认证摊销是下一个数量级瓶颈”这一判断提供了较强证据。与此同时，其收益依赖于客户端在线参与、BLS 聚合签名假设和较复杂的工程实现；信任模型差异与故障场景覆盖也提醒读者，跨系统倍数的解读应结合具体部署条件。

总体而言，这是一篇问题定位精准、协议设计完整、系统实现扎实、实验规模突出的高吞吐拜占庭原子广播工作。其最重要的遗产可能不只是 43.6M op/s 这一数字，而是把研究社区的优化焦点从“如何更快排序”推进到“如何更便宜地认证与去重”，并给出了一个可退化、可工程化的可行答案。

（完）
