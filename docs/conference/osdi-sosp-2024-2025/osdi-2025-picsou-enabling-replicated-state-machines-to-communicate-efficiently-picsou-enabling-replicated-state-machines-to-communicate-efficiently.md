---
title: "Picsou: Enabling Replicated State Machines to Communicate Efficiently"
title_zh: Picsou：让副本状态机高效通信
authors: "Reginald Frank, Micah Murray, Chawinphat Tankuranand, Junseo Yoo, Ethan Xu, Natacha Crooks, Suyash Gupta, Manos Kapritsos"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-frank.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: 提出跨集群一致广播新原语，使副本状态机之间高效通信
tldr: 副本状态机之间目前缺乏形式化框架与高效协议，难以进行有效通信。论文提出跨集群一致广播原语C3B并实现Picsou，借鉴网络与TCP思想，使两个状态机在无故障时以常量元数据开销通信、在故障时仅需少量消息重传。其核心是仲裁确认Quacks机制，让节点精确判定消息何时可交付，并同时支持崩溃容错与拜占庭容错协议。该工作为分布式系统提供了一种通用高效的通信原语，与集合通信协议设计主题有方法层面的关联。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
motivation: 副本状态机之间缺少形式化框架和高效协议，跨集群通信开销大。
method: 提出跨集群一致广播原语C3B及实现Picsou，基于仲裁确认机制控制消息交付。
result: 无故障时元数据开销恒定，故障时仅需最少量的消息重传。
conclusion: 为分布式系统提供通用且兼容崩溃与拜占庭容错的通信原语。
---

## Abstract
Replicated state machines (RSMs) cannot communicate effectively today as there is no formal framework or efficient protocol to do so. To address this issue, we introduce a new primitive, Cross-Cluster Consistent Broadcast (C3B) and present Picsou, a practical C3B implementation. Picsou draws inspiration from networking and TCP to allow two RSMs to communicate with constant metadata overhead in the failure-free case and a minimal number of message resends in the case of failures. Picsou is flexible and allows both crash fault tolerant and Byzantine fault tolerant protocols to communicate. At the heart of Picsou's good performance and generality is the concept of Quacks (quorum acknowledgments). Quacks allow nodes in each RSM to precisely determine when messages have definitely been received, or likely lost. Our results are promising: we obtain up to 24× better performance than prior solutions on microbenchmarks and applications, ranging from disaster recovery to data reconciliation.

---

## 论文详细总结（自动生成）

# Picsou 论文结构化总结

## 1. 核心问题与研究动机

- **背景**：现代组织广泛使用基于共识协议的副本状态机（RSM）来提供可靠性、故障隔离与灾难恢复，例如 etcd、Kubernetes、微服务、区块链等。这些 RSM 经常需要**跨集群通信**（如 etcd 灾难恢复镜像、跨机构数据共享与对账、区块链互操作/跨链桥）。
- **核心问题**：当前 RSM 之间**缺乏形式化框架与高效通信协议**。现有方案存在明显缺陷：
  - 临时性方案（ad-hoc）缺乏保证，或依赖可信第三方（如 Kafka 内部再跑一个 Raft 集群）；
  - 全对全广播（All-to-All）复杂度为 O(ns×nr)，在广域网下带宽成本极高；
  - 部分方案（如区块链桥）形式化保证薄弱，曾导致重大金融攻击。
- **需求归纳**：任何 RSM 间通信方案应满足四点——(1) **强保证**（可形式化讨论）；(2) **故障鲁棒性**（崩溃或恶意节点不影响正确性、不显著降低吞吐）；(3) **常见情况低开销**（无故障时单条消息 + 常量元数据）；(4) **通用性**（支持异构规模、通信模型与故障模型，如 CFT 与 BFT 互通）。

## 2. 方法论

### 2.1 核心贡献：C3B 原语
- 论文首先提出 **Cross-Cluster Consistent Broadcast (C3B)** 原语，将可靠广播（Reliable Broadcast）推广到"集群到集群"场景。
- **形式化定义**：定义 `Transmit`（发送方 RSM 中某正确副本调用 C3B）与 `Deliver`（接收方 RSM 中某正确副本输出消息）两个粗粒度操作。
- **两条正确性属性**：
  - **最终交付（Eventual Delivery）**：若 Rs 向 Rr 传输 m，则 Rr 最终会交付 m。
  - **完整性（Integrity）**：Rr 交付 m 当且仅当 Rs 确实传输过 m。
- 注意：C3B 只要求接收方**至少一个正确副本**收到，不要求全部收到，以保持最大灵活性；可通过接收方内部广播或共识来强化。

### 2.2 系统模型
- 采用 **UpRight 故障模型**统一刻画崩溃与拜占庭故障：拜占庭节点可发生"commission failure"（主动偏离协议），其他故障节点只发生"omission failure"（不发/不收消息）。
- 用统一公式 `2u + r + 1` 描述 RSM 规模：设 `u = r = f` 得到 `3f+1` 的 BFT 系统；设 `r = 0` 得到 `2f+1` 的 CFT 系统。
- 支持**份额（share/stake）**：以 δᵢⱼ 表示副本权重，可刻画 PoS 系统（如 Algorand）。
- 不假设同步或部分同步，只假设消息最终送达，且接收方 RSM 能验证发送方确实提交了某交易。

### 2.3 Picsou 协议设计（C3B 的具体实现）

**核心思想：借鉴 TCP 的三大机制并适配多对多、拜占庭场景**
- 全双工通信 + 累积确认（cumulative ACK）→ 低开销；
- 重复 ACK 揭示消息丢失 → 故障检测；
- 但需解决 TCP 未考虑的问题：多对多传播、恶意节点可能谎报、不能因恶意节点触发虚假重传。

**关键创新：QUACK（Quorum Acknowledgment，仲裁确认）**
- 接收方维护已收消息的有序列表，计算"连续收到的最高序号 p"，发送 `ACK(p)` 累积确认 1..p。
- 发送方维护长度为 nr 的数组，记录来自每个接收副本的最高 ACK；当**至少 u+1 个副本**确认了到 p 为止的所有消息时，形成 **QUACK**，证明至少一个正确副本已收到（因最多 u 个故障副本）。
- 在拜占庭场景下，ACK 用 MAC 保护；重复 QUACK 需 **r+1** 个才触发重传（防止恶意节点单方面引发虚假重传）。

**发送调度**
- 消息格式 `⟨m, k, k'⟩_Qs`：k 为共识提交序号，k' 为 Picsou 流内序号（k'=⊥ 表示不传输）。
- **轮询分区 + 收发方轮转**：副本 Rˢₗ 发送满足 `k' mod ns ≡ l` 的消息；每次发送后接收方索引 `J ≡ j+1 mod nr` 轮转。节点 ID 由可验证随机源分配，防止恶意节点选择位置。
- 目标：无故障时每条消息只发一次、每集群 O(n) 副本、元数据为常量。

**失败检测与重传**
- 若某副本已 QUACK 到 k，却又收到重复 `ACK(k)`，说明有正确副本未收到 k+1 → 判定消息丢失。
- **重传者选举**（无需额外通信）：`sender_new = (sender_original + #retransmit) mod ns`，各正确副本独立计算，ID 匹配者负责重传，每轮仅一个发送者。
- **并行累积确认（φ-list）**：解决"顺序恢复"瓶颈。接收方在累积 ACK 之外附带一个 φ 位列表，描述累积 ACK 之后最多 φ 条消息的投递状态（每消息 1 bit，可压缩/布隆过滤）。发送方可并行形成 φ 个 QUACK 并并行重传，避免串行恢复导致的长尾延迟。

**垃圾回收（GC）**
- 直接 GC 可能导致停顿（故障副本可能只向 ur+1 个副本（其中 ur 个故障）广播后消失）。解决方案：收到过期的重复 QUACK 时，在元数据中附带自身最高 QUACK 序号 k；接收方收到 rs+1 个此类消息后可推进累积计数器或从其他副本取回消息。

**重配置**
- 假设存在配置服务维护成员关系（业界标准做法），要求同一消息的所有 ACK 属于同一视图且达到阈值；重配置后仅重传未获仲裁确认的消息。

**加权 RSM（Stake）支持**
- **加权 QUACK**：ACK 带权重 δ，总权重达到 ui+1 即形成 QUACK。
- **动态份额调度器（DSS）**：轮询在 stake 不均时严重失衡（如 33% stake 节点承担 50% 流量）；彩票调度短期波动大。DSS 借鉴 Linux CFS 与**汉密尔顿分配法（Hamilton's method of apportionment）**：
  - 计算标准除数 `SD = Δ/q`、标准配额 `SQ = δ/SD`、下配额 `LQ = floor(SQ)`、惩罚比 `PR = SQ - LQ`；
  - 先分配 LQ，剩余名额按 PR 从高到低依次 +1，保证短期与长期公平且保留并行性。
- **重传时的 stake 缩放**：两 RSM 总 stake 差异巨大时（如 Δs=4, Δr=4,000,000），所需重传数会爆炸。Picsou 在首次收到重复 QUACK 时按**最小公倍数（LCM）** 等比放大双方 stake，使所需重传数与相对 stake 比例解耦，且仅故障时才缩放，保持常见情况小量子。

## 3. 实验设计

### 3.1 测试平台与 RSM
- 实现：约 **4500 行 C++20**，Protobuf v3.10.0 序列化，NNG v1.5.2 网络库，设计为即插即用库。
- 四类 RSM：
  1. **File**：无限快生成已提交消息的内存文件，用于饱和压测 C3B 协议上限；
  2. **Raft**：etcd v3.0 的 CFT 实现；
  3. **ResilientDB**：高性能 PBFT 实现（BFT）；
  4. **Algorand**：PoS 区块链协议。

### 3.2 对比基线（5 个）
- **OST（One-Shot）**：单发单收，仅作性能上界，不满足 C3B；
- **ATA（All-to-All）**：O(ns×nr) 全广播；
- **LL（Leader-to-Leader）**：领导者对领导者，领导者故障时无保证；
- **KAFKA**：工业标准（2.13-3.7.0），内部用 Raft；
- **OTU**：GeoBFT 的跨 RSM 协议，领导者发给至少 ur+1 个接收副本。

### 3.3 实验场景
- **微基准**：副本数 4–19、消息大小 0.1 kB–1 MB、φ-list 大小（0–256）、stake 分布变化、地理复制（US-West vs 香港，170 Mbit/s，RTT 133ms）。
- **故障实验**：33% 崩溃故障；拜占庭故障（选择性丢包、错误 ACK：Picsou-Inf / Picsou-0 / Picsou-Delay）。
- **真实应用**：(1) etcd 灾难恢复（跨数据中心单向镜像）；(2) 数据共享与对账（双向，机构 A/B 各自 etcd）；(3) DeFi 区块链桥（Algorand↔Algorand、ResilientDB↔ResilientDB、Algorand↔ResilientDB）。

### 3.4 环境
- GCP **c2-standard-8**（Intel Cascade Lake，8 vCPU，32 GiB RAM，15 Gbit/s），最多 **45 台节点**；每实验运行 180 秒（含 30 秒预热/冷却）。

## 4. 资源与算力

- 本文**不涉及机器学习训练，未使用 GPU**，也未报告训练时长等算力指标。
- 计算资源仅为分布式系统实验所需的 **GCP c2-standard-8 云主机，最多 45 台**（CPU 型实例）。
- 这一资源规模对于系统类论文属正常水平，作者未额外讨论算力开销。

## 5. 实验数量与充分性

**实验组数（大致统计）**：
- 微基准：副本规模扫描（2 种消息大小）、消息大小扫描（2 种网络规模）、stake 分布（多档 Picsou1–Picsou64）、地理复制（2 地、多规模）≈ **4 大类，每组 4–19 个数据点**；
- 故障实验：崩溃故障、φ-list 扫描（φ0–φ256 共 6 档）、三种错误 ACK 攻击 ≈ **3 组**；
- 应用实验：灾难恢复、数据对账、DeFi 跨链 ≈ **3 组**，其中跨链含 3 种子场景。
- 合计约 **10+ 组主要实验**，覆盖性能、故障、应用三个维度。

**充分性与客观性评估**：
- **优点**：基线选择全面（含性能上界 OST、工业标准 Kafka、BFT 分片方案 OTU），既有微基准又有真实应用；故障注入覆盖崩溃与多类拜占庭攻击；报告了 φ-list 的敏感性分析（消融）；地理广域网场景贴近实际部署。
- **潜在不足**：故障实验主要固定 33% 故障比例，未系统扫描故障比例；拜占庭攻击类型虽列四类但明确排除 DDoS（认为超出范围）；应用实验节点规模较小（如 Kafka 仅 3 节点，作者承认受已知高延迟消费者问题影响，结果偏悲观）；部分实验（如 stake 实验）中高 stake 节点成为瓶颈的结论较定性。

## 6. 主要结论与发现

- **性能显著提升**：微基准中，当共识非瓶颈时，Picsou 比 ATA 在小网络（4 节点）快 **3.2×**，大网络（19 节点）快最高 **24×**；地理场景下比 ATA 快 **12×（n=4）至 44×（n=19）**。
- **真实应用**：灾难恢复与数据对账中，Picsou 比 Kafka 快约 **2×**；Picsou 通过分片发送充分利用多链路带宽（5 节点 × 50 MB/s ≈ 250 MB/s），可饱和 Raft 磁盘吞吐（70 MB/s），而 ATA/OTU/LL 受限于单链路或领导者。
- **故障鲁棒**：33% 崩溃故障下性能仅下降 **22.8%–30.5%**，仍比 ATA/OTU/LL 快至少 2×（大网络达 8.9×）；错误 ACK 攻击危害远小于崩溃，因正确节点本就预期 u 个 ACK 可能是谎言，仅暂时延迟仲裁形成。
- **通用性验证**：成功打通 PBFT（ResilientDB）、Raft、Algorand 等异构协议；跨链场景吞吐下降 <15%，且与节点 stake 无关；Picsou 能处理两 RSM 间巨大吞吐差异（慢的 Algorand 与快的 ResilientDB 互通）。
- **理论界**：同步期最多重传 us+ur+1 次；附录证明重传 8 次即达 99% 交付概率，72 次达 1−10⁻⁹ 概率。

## 7. 优点（亮点）

- **问题定位精准**：首次形式化"RSM 间通信"为 C3B 原语，填补了可靠广播只关注"组内"、忽视"组间"的空白。
- **TCP 类比的巧妙迁移**：全双工、累积 ACK、重复 ACK 检测丢失等经典机制被成功适配到多对多 + 拜占庭场景，QUACK 概念简洁而强大。
- **常量元数据 + 单次发送**：无故障时仅 2 个额外计数器，符合高效通信的核心诉求。
- **统一故障模型**：用 UpRight 模型 + `2u+r+1` 公式统一 CFT/BFT，协议逻辑无需为两种模式分别设计，通用性强。
- **拜占庭鲁棒性设计细致**：MAC 保护 ACK、r+1 重复 QUACK 阈值、随机化节点 ID、重传者选举公式，系统性防止恶意节点引发虚假重传。
- **工程与理论结合**：既有完整实现（4500 行 C++）与真实应用评测，又有形式化正确性证明与概率界（附录）。
- **Stake 支持考虑周全**：用汉密尔顿分配法解决无界 stake 下的公平调度，用 LCM 缩放解决 stake 悬殊时的重传爆炸，思路新颖。

## 8. 不足与局限

- **异步系统固有限制**：无法区分"消息丢失"与"消息延迟"，

- **异步系统固有限制**：无法区分"消息丢失"与"消息延迟"，重传触发与终止只能依赖概率界而非确定性判据。在网络抖动、长尾延迟或短暂分区下，重复 QUACK 可能被误判为丢失，导致冗余重传；论文虽给出"重传 8 次达 99%、72 次达 1−10⁻⁹"的概率界，但该界隐含"每次重传独立丢失"的假设，而真实广域网丢包常呈突发相关性（burst loss），实际收敛速度可能慢于理论值。
- **跨集群验证假设成本未量化**：模型假设接收方 RSM 能验证发送方确实提交了某交易，这在同构系统内可行，但异构链之间通常需要轻客户端证明、状态证明或中继签名，其带宽与验证开销可能抵消 Picsou"常量元数据"的优势。论文对此仅作假设，未给出具体实现与测量。
- **外部配置服务的保证缺口**：重配置依赖一个外部配置服务，但该服务自身的崩溃/拜占庭行为未被纳入 C3B 的正确性论证；视图切换期间"同一消息的所有 ACK 须属同一视图"的约束在并发重配置下的安全性论证较简略。
- **故障模型与密钥管理**：UpRight 模型区分 commission/omission failure 较为新颖，但两类故障在实际部署中的判定边界、以及 MAC 密钥的分发、轮换与吊销机制未展开；拜占庭安全性在密钥泄露场景下的退化未讨论。
- **攻击面覆盖有限**：作者明确将 DDoS 排除在范围外，也未考虑网络层攻击（日食攻击、路由劫持）与多恶意节点**合谋**构造边界 QUACK 的情形；错误 ACK 实验仅测试单点或少量恶意副本，未系统评估接近阈值（u 个恶意副本）时的性能退化曲线。
- **实验覆盖的不足**：
  - 故障实验基本固定 33% 故障比例，未扫描故障比例从 0 到阈值的连续变化；
  - 应用实验规模偏小（Kafka 仅 3 节点、跨链 3–5 节点），作者也承认 Kafka 对比结果因已知消费者高延迟问题而偏悲观；
  - 广域网只测 US-West↔香港一条链路，未覆盖多区域、高抖动、非对称带宽等更复杂拓扑；
  - DSS 调度器与 LCM stake 缩放在大规模（>19 节点）与频繁 churn 下的表现未验证，极端 stake 比例（如 1:10⁶）下的消息放大与内存开销缺乏定量分析。
- **理论分析的形式化程度**：正确性证明与概率界主要放在附录，未使用 TLA+、Coq、Isabelle 等机器验证工具；对活性（liveness）在部分同步假设下的讨论不如安全性充分。
- **工程成熟度**：4500 行 C++ 原型偏研究性质，未讨论持久化、监控、背压、密钥轮换等生产级特性；论文未明确说明代码与实验数据是否开源，影响可复现性。
- **评估维度的缺口**：主要报告吞吐，对延迟分布（p99、尾延迟）、CPU/内存开销、ACK 元数据在长流下的摊销成本报告较少；φ-list 的位图/布隆过滤压缩带来的假阳性对重传行为的影响未深入分析。

## 9. 总体评价与启示

- **定位与贡献**：这是一篇"问题定义 + 原语 + 协议 + 实现 + 评测"完整的系统类论文。其最大价值不在于单项工程优化，而在于**首次把"RSM 间通信"抽象为 C3B 原语**，并给出一个在常见情况下开销接近理论下界（每条消息一次发送、常量元数据）的通用协议。这一抽象有望成为后续跨集群/跨链通信工作的公共语言与比较基准。
- **方法论启示**：将 TCP 的"累积确认 + 重复 ACK 检测丢失"迁移到多对多、拜占庭环境，并通过 QUACK 阈值（u+1 形成、r+1 触发重传）把"正确性"与"抗恶意"解耦，是一个干净且可复用的设计范式；重传者选举公式与随机化节点 ID 则展示了"用确定性规则替代额外通信"的工程智慧。
- **适用边界**：Picsou 的优势在"共识非瓶颈、消息流持续、接收方集群有一定规模"的场景最明显；当集群极小（如 1–2 节点）、消息极稀疏或网络极不稳定时，其相对 ATA/LL 的收益会收窄，甚至元数据开销占比上升。
- **后续可研究方向**：(1) 将配置服务纳入形式化模型，实现端到端的重配置安全性证明；(2) 引入相关性丢包模型修正重传概率界；(3) 在真实异构链上实现并测量跨集群验证（轻客户端证明）的端到端成本；(4) 研究多恶意节点合谋下的 QUACK 阈值优化与自适应 φ 调节；(5) 与 DAG/ gossip 类广播方案做系统对比。

（完）
