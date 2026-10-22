---
title: "{FineMem}: Breaking the Allocation Overhead vs. Memory Waste Dilemma in {Fine-Grained} Disaggregated Memory Management"
title_zh: FineMem：破解细粒度解耦内存管理中的分配开销与内存浪费两难
authors: "Xiaoyang Wang, Yongkun Li, Kan Wu, Wenzhe Zhu, Yuqi Li, Yinlong Xu"
date: 2025
publication_date: 2025
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi25-wang-xiaoyang.pdf"
tags: ["query:dcn-comm"]
score: 6.0
evidence: 面向数据中心内存解耦的RDMA远程内存管理
tldr: RDMA内存解耦是降低数据中心内存成本的重要手段，但远程内存分配存在延迟与可扩展性难题，导致现有系统只能采用GB级粗粒度分配并造成内存浪费。FineMem提出RDMA连接的远程内存管理系统，通过按计算节点预注册内存区域把注册开销移出分配路径，并保证远程内存的可用性。系统实现了高性能细粒度分配，缓解了分配开销与内存浪费的两难。其工作直接服务于低延迟RDMA数据中心通信基础设施。
source: OSDI-2025-USENIX
selection_source: conference_retrieval
motivation: RDMA内存解耦下远程内存分配开销大，迫使系统采用粗粒度分配并造成浪费。
method: 通过按计算节点预注册内存区域，将RDMA注册成本移出分配路径，实现细粒度分配。
result: 在保持低延迟的同时实现了高性能细粒度远程内存分配。
conclusion: 为RDMA数据中心内存解耦提供了可扩展的细粒度管理方案。
---

## Abstract
RDMA-enabled memory disaggregation has emerged as an attractive approach to reducing memory costs in modern data centers. While RDMA enables efficient remote read/write operations, it presents challenges in remote memory (de)allocation. Consequently, existing systems adopt coarse-grained allocations (in GBs), leading to memory waste. We introduce FineMem, an RDMA-connected remote memory management system that enables high-performance, fine-grained memory allocation. FineMem addresses latency and scalability challenges related to fine-grained allocations. It removes RDMA memory region (MR) registration costs from allocation paths through per-compute node MR pre-registration, while ensuring remote memory isolation using RDMA memory windows and a trusted allocation service on each compute node. It employs a lock-free, one-sided RDMA-based protocol to allocate memory chunks (e.g., 4KB, 2MB) without involving the memory node's CPU and maintains metadata consistency during compute node failures via logging. We show that FineMem reduces remote memory allocation latency by as much as 95% compared to state-of-the-art remote memory management systems. It enables memory malloc systems, key-value stores systems, and swap systems running on FineMem to achieve low memory waste with minimal overhead.

---

## 论文详细总结（自动生成）

# FineMem 论文总结

## 一、核心问题与整体含义（研究动机与背景）

- **背景**：RDMA 支持的内存解耦（memory disaggregation）是降低现代数据中心内存成本的重要架构方向，计算节点可通过单边 RDMA 操作直接读写远端内存，绕过内存节点 CPU。
- **核心痛点**：RDMA 的数据访问虽高效，但远程内存的**分配与释放**极其昂贵：
  - MR（Memory Region）注册需在内存节点 CPU 上完成物理页固定、RNIC 页表更新等操作，**注册一个 4MB 区域耗时超过 480 μs**；
  - RPC 式分配受限于内存节点有限算力，客户端增多后迅速成为瓶颈（如 FUSEE 在 32 客户端后吞吐饱和）；
  - 现有单边方案（如 CXL-SHM 式自由数组）因元数据设计不佳，CAS 重试频繁，延迟不可预测。
- **后果——"分配开销 vs. 内存浪费"两难**：现有系统被迫采用 GB 级粗粒度分配来摊销开销，导致内存严重浪费（实验显示从 1GB 降到 2MB 会使 KV 吞吐下降约 17%；预映射方案内存利用率常低于 50%）。
- **论文目标**：在 RDMA 解耦内存池中实现**低开销的细粒度分配（4KB、2MB 等）**，同时保证多系统共享下的隔离性、可扩展性与崩溃一致性，从而降低数据中心内存成本。

## 二、方法论

### 核心思想
用**单边 RDMA** 完成分配/释放，彻底绕开内存节点 CPU；通过"预注册 + MW 能力 + 可信分配服务"解决隔离问题，通过"两层位图 + 竞争控制"解决并发分配效率问题，通过"紧凑提交点 + 临时日志"解决崩溃一致性问题。

### 关键技术细节

1. **移除 MR 注册延迟（隔离挑战的解决）**
   - 内存节点启动时把整个远端内存预注册为单个 MR，按计算节点预映射。
   - 用 **RDMA Memory Window（MW）** 实现细粒度访问隔离：每个 chunk/span/section 预绑定 MW 与 rkey，rkey 生成约 1 μs（对比 MR 注册数百 μs），单 ConnectX-6 NIC 支持约 1600 万 MW 条目，可通过多 VF（最多 128 个）扩展 64GB 上限。
   - 预生成主 rkey 与备份 rkey，释放时用一次单边 CAS 将主 rkey 替换为备份 rkey，把 rkey 再生移出关键路径；后台线程异步失效旧 rkey 并生成新 rkey。

2. **两层位图树（高效可扩展分配）**
   - 地址空间分两层：**Section**（含 16 个连续 span，每个 span 128KB）与 **Span**（含 32 个连续 chunk）。
   - Section 用 32 位位图，每 span 2 bit 表示状态：`00` 全空、`11` 全满、`01` 正常使用、`10` 存在竞争。
   - Span 用 32 位 free map 跟踪 32 个 chunk；64 位头部对齐 RDMA CAS 操作宽度，剩余位用于竞争控制与日志。
   - 分配流程：先读 section 头定位可用 span；大于 128KB 的分配用一次 CAS 直接置位；否则读 span 头，标记空闲位并用一次 CAS 更新。大分配跨多个 section 时需多次 CAS。
   - 每个计算节点缓存最多 64 个 section/span 元数据（共 512 字节），批量读取以摊销往返开销。

3. **竞争控制（减少 CAS 重试）**
   - 每个分配头记录连续 CAS 失败次数；超过高阈值（如 10）标记为 `contended(10)`，低于低阈值（如 3）恢复为 `normal(01)`。
   - 后续分配按 `normal(01) > empty(00) > contended(10)` 优先级选择，实现反压式退避，降低重试与尾延迟。

4. **崩溃一致性（计算节点失效）**
   - 以**位图 CAS 成功**作为提交点，并把临时 redo-log 紧凑嵌入 64 位头部：chunk 偏移 5 bit、size 3 bit、timestamp 7 bit、user ID 14 bit。
   - 7 位时间戳通过"仅在 malloc/free 方向切换时递增"来延缓溢出，可检测约 1 秒内的过期更新。
   - 提交后由任意线程乐观地将临时日志刷入完整 redo-log，并比较时间戳决定刷新或丢弃；跨层不一致时用 span 的 `in_use` 位校正 section 位图。
   - 恢复流程：扫描 section/span 元数据刷新临时日志 → 扫描完整日志回收崩溃节点的 chunk → 再生 rkey；内存节点每秒发送心跳检测计算节点存活。

5. **分配服务（元数据保护）**
   - 每个计算节点运行可信分配服务，只有它持有读写元数据区所需的 rkey；DM 系统通过 IPC 调用 `malloc(size)`/`free(addr)`，IPC 开销约 2–10 μs。
   - 每个 CPU 核一个 worker 线程，通过共享信号量同步。
   - API 保护：malloc 受每系统配额限制，free 校验 rkey 正确性；二进制保护：外部私钥授权，仅可信服务可与内存节点建立管理连接（多租户场景可部署在 hypervisor 层）。

## 三、实验设计

- **测试平台**：CloudLab 集群，**16 个计算节点 + 1 个内存节点**；节点为 Intel Xeon 8360Y CPU、Mellanox ConnectX-6 100Gb NIC、256GB 内存；Ubuntu 22.04 + Linux 5.15 + Mellanox OFED 5.8。计算节点最多用 32 核，内存节点限制为单核。
- **实现规模**：FineMem 本体 C++ 约 8.5k LOC；FineMem-User（基于 mimalloc）约 1.5k LOC；FineMem-Swap（移植 FastSwap）约 0.7k LOC；FineMem-KV 仅改 FUSEE 慢路径约 300 LOC。
- **对比基线**：
  - `Premmap-One-sided`（源自 CXL-SHM，自由数组搜索）；
  - `Premmap-RPC`（源自 FUSEE/Patronus，内存节点侧 memkind 分配器）；
  - `OnDemand-RPC`（运行时 MR 注册的细粒度分配）；
  - 基线统一预映射 200GB 以排除注册开销干扰。
- **基准与场景**：
  1. **分配微基准**：基于 Redis 测试套件的块分配/释放负载（4KB–2MB，初始 64K 块，共约 100 万次分配，重复 500 轮），扫描分配粒度、客户端线程数（至 512 线程）、内存节点数。
  2. **因子分析（消融）**：分别评估两层位图、竞争控制、rkey 能力读取、redo-log 刷新、服务层 IPC 的延迟贡献。
  3. **混合大小分配 trace**：采集 jemalloc、mimalloc、tcmalloc、ptmalloc 在 Shbench、Larson、Rptest、Lean 上的系统调用序列，跨 16 节点 × 8 线程重放。
  4. **内存 malloc 系统**：ThreadTest、Shbench、Larson，16 节点 × 32 线程。
  5. **KV 存储**：YCSB A/B/C/D 负载（KV 对 512B，块大小 4KB 与 2MB）。
  6. **Swap 系统**：XGBoost、Snappy、Redis 混合负载，5 计算节点（80GB 本地内存）+ 2 内存节点（160GB DM）；另随机生成 500 组小规模与 500 组大规模工作负载。
  7. **多系统共享**：8 节点跑 KV + 8 节点跑 malloc 微基准并发，评估相互干扰。
  8. **恢复能力**：时间戳溢出窗口、恢复速度、恢复对并发的影响。

## 四、资源与算力

- **本文为系统类论文，未使用 GPU，也未涉及模型训练**，因此不存在 GPU 型号、数量或训练时长等信息。
- 实验资源为 CloudLab 集群的 **16 个计算节点 + 1 个内存节点**（swap 实验扩展为 5 计算节点 + 2 内存节点；大规模仿真为 40 节点共享 1280GB DM 的模拟场景）；单节点 32 核可用、256GB 内存、100Gb RDMA NIC。
- 文中未报告总机时消耗，也未说明实验运行的总时长。

## 五、实验数量与充分性

- **实验组数**：覆盖约 8 类实验（粒度扫描、线程扩展性、多内存节点、因子消融、混合大小 trace、三类系统案例、跨系统共享、恢复能力），加上多种基线（3 个）与多种应用（3 类系统 × 多个 benchmark），总体规模较充分。
- **充分性亮点**：
  - 微基准每个配置重复 10 次取稳定结果；
  - 同时报告平均与 P99/最大值延迟，并统计 CAS 重试次数，指标较全面；
  - 消融实验定量拆解各设计组件的延迟贡献（两层位图约降 52.5%，竞争控制再降 44%，合计约 95%）；
  - 混合大小 trace 基于真实分配器采集，提升外部有效性；
  - 跨系统共享实验直接回应"多系统共享同一内存池"的核心动机。
- **公平性**：基线在同一代码库内移植、统一预映射 200GB，控制变量较严谨。
- **潜在不足**：多数实验在单一 CloudLab 集群、单一 NIC 型号（ConnectX-6）上完成；大规模 swap 场景（40 节点、1280GB）为**模拟/仿真**而非真实部署；未与其他真实生产级解耦内存系统做端到端对比。

## 六、主要结论与发现

- FineMem 将远程内存分配延迟相比最先进方案**降低最多 95%**；4KB 粒度、512 线程下平均延迟 43.2 μs（P99 79.3 μs），而 Premmap-One-sided 为 763 μs（P99 达 16 ms）。
- 平均 CAS 重试次数从 45.1 降至 1.33，尾延迟可预测性大幅改善。
- 内存 malloc 系统场景下，内存利用率相比粗粒度预映射方案**提升 2.25×–2.8×**，额外开销仅 **2.5%–4.1%**。
- KV 存储在 YCSB-A 下带宽提升约 **27%–110%**；4KB 块可达到接近 2MB 块的带宽，同时内存成本降低约 45%。
- Swap 系统内存利用率从 41.39% 提升至 **74.06%**，作业吞吐提升 **17.71%**；随机工作负载下平均吞吐提升 8.38%–10.69%。
- 多系统共享时 FineMem 保持性能稳定，而 Premmap-RPC 带宽下降 46.8%、Premmap-One-sided 下降 75.5%。
- 内存节点侧开销可控：rkey 再生线程约占单核 15%，元数据仅占内存约 0.4%（每 100GB 约 400MB）。

## 七、优点

- **问题定位精准**：清晰刻画了"注册开销 vs. 内存浪费"的真实两难，并用定量数据（吞吐、利用率、碎片）支撑动机。
- **设计组合巧妙**：
  - 用 MW 替代 MR 实现"低成本能力隔离"，并把 rkey 再生移出关键路径（主/备 rkey + CAS 替换）；
  - 两层位图把分配 CAS 次数压到平均 ≤2 次，且 2-bit 状态同时编码"满/空/竞争"，一物多用；
  - 把临时 redo-log 压缩进 64 位头部，使"提交点 + 日志 + 时间戳"合并为一次 CAS，兼顾一致性与性能。
- **工程完整度高**：同时移植到 malloc 系统、KV 存储、内核 swap 三类系统，证明通用性；代码开源。
- **实验严谨**：基线同库移植、统一预映射、重复 10 次、平均与尾延迟并报、消融定量拆解。
- **安全考量周全**：API 配额、rkey 校验、私钥授权、hypervisor 层部署建议，覆盖对抗场景。

## 八、不足与局限

- **平台依赖性强**：深度依赖 RDMA 专有机制（MW、rkey、单边 CAS），向 CXL 等新兴互连迁移需额外设计（缓存、MPK 绑定等机制差异），论文自认尚未完成。
- **未覆盖的问题**：不解决缓存一致性、多内存节点间的数据冗余/容错；仅处理计算节点崩溃一致性，未讨论内存节点自身故障。
- **预注册全量内存的约束**：需要预先与内存节点指定服务

- **预注册全量内存的约束**：需要在系统启动时与内存节点协商并一次性预注册全部内存，且按计算节点预映射；这意味着内存池的成员关系、容量划分在运行期较难弹性调整，与"按需加入/退出解耦池"的云化运维模式存在张力。
- **MW 条目数量上限**：单 NIC 约 1600 万 MW 条目、单 VF 约 64GB 的可寻址上限，需要通过多 VF（最多 128 个）横向扩展；在超大规模（TB 级以上、数千租户）部署时，rkey 空间管理与 VF 映射本身会成为新的管理复杂度来源。
- **元数据扫描成本随规模线性增长**：恢复流程与后台日志刷新都需扫描 section/span 元数据，虽然元数据占比仅约 0.4%，但在超大内存池（如 1280GB 级）上扫描耗时与对前台请求的干扰论文只做了有限评估，且该场景部分依赖仿真。
- **竞争控制阈值偏经验性**：高/低阈值（如 10/3）与"连续失败计数"策略缺少理论模型或自适应机制，在负载模式剧烈变化时是否仍最优未充分验证；论文未给出阈值敏感性扫描的完整结果。
- **时间戳窗口有限**：7 位时间戳通过"方向切换才递增"延缓溢出，仅能检测约 1 秒内的过期更新；在极端长尾或节点长时间无操作后突发的场景下，正确性依赖恢复扫描与 `in_use` 位校正兜底，边界条件的鲁棒性论证偏弱。
- **服务层引入的额外一跳**：分配需经每节点可信服务的 IPC（约 2–10 μs），相对纯单边路径仍有固定开销；论文的因子分析给出了该项占比，但未讨论在极低延迟敏感型应用（如纳秒级 RPC 中间件）中的可接受性。
- **安全模型假设较强**：二进制保护依赖外部私钥授权与"仅可信服务可建立管理连接"，未对恶意计算节点伪造 rkey、重放 CAS、耗尽配额等攻击做系统性威胁建模与实测；多租户下建议部署于 hypervisor 层，但未给出实现与开销评估。
- **故障模型不完整**：仅覆盖计算节点崩溃一致性，未处理内存节点自身故障、NIC 故障、网络分区下的元数据一致性；也未讨论多内存节点间的数据冗余与容错，这与"解耦内存作为数据中心基础设施"的可靠性诉求仍有距离。
- **对比基线范围有限**：三个基线均在同一代码库内移植并统一预映射 200GB，控制了变量但也意味着缺少与真实生产级解耦内存系统（如商业 CXL 内存池方案、其他 RDMA 池化系统的最新版本）的端到端对比；部分大规模结论来自仿真。
- **硬件与生态单一**：实验集中在单一 CloudLab 集群、单一 NIC 型号（ConnectX-6）与单一内核版本；MW、单边 CAS 等语义在其它厂商 RNIC 上的行为差异未评估，结论的可移植性有待验证。
- **应用覆盖偏"内存分配接口"**：三类案例系统均以 malloc/free 或页交换为接入点，对需要对象级共享、跨节点指针、持久化语义的上层系统（如分布式数据结构、持久内存数据库）适配性未讨论。

## 九、对领域的启发与可推广性

- **"把能力（capability）而非权限位作为隔离原语"**：用 MW/rkey 把"访问权"变成可低成本生成、可原子替换的一等对象，这一思路可推广到其它需要高频细粒度权限变更的场景（如多租户共享缓存、函数级沙箱内存）。
- **"把提交点、日志与元数据压缩进一个原子字"**：将 redo-log 压缩嵌入 64 位头部的做法，为其它单边可写共享结构的崩溃一致性提供了可复用的模式——用一次 CAS 同时完成"生效 + 可恢复性记录"。
- **"用状态编码做反压"**：把竞争程度编码进 2-bit 状态位并影响后续选择顺序，是一种零额外通信的自适应退避机制，可迁移到其它基于 CAS 的分布式分配器/计数器。
- **对内存解耦生态的意义**：论文定量说明"分配开销"而非"数据通路带宽"才是阻碍细粒度的关键瓶颈，这一结论对 CXL 内存池、池化内存的操作系统接口设计具有直接参考价值。

## 十、总体评价

FineMem 是一篇问题定义清晰、设计取舍自洽、工程落地完整的系统论文。其核心贡献不在于单点技巧，而在于把"单边 RDMA 上的细粒度分配"拆解为隔离、并发、一致性三个子问题，并分别给出低成本、可组合的机制：MW 主/备 rkey 解决隔离与再生延迟，两层位图 + 竞争状态位把 CAS 次数压到接近理论下限，紧凑临时日志使崩溃恢复与快路径共用一次原子操作。实验在粒度、线程数、应用类型、跨系统共享与恢复等多个维度展开，基线与消融控制较为严谨，结论（延迟降幅最高 95%、利用率提升 2.25×–2.8×、KV 带宽提升 27%–110%）有数据支撑且与动机形成闭环。

主要短板在于平台与故障模型的覆盖边界：强依赖 RDMA 专有语义、未处理内存节点故障、大规模结论部分依赖仿真、安全威胁模型偏假设性。这些并不削弱其作为"细粒度解耦内存分配"方向代表作的价值，但提示后续工作需在跨互连（CXL）可移植性、池级容错与多租户对抗性验证上继续推进。总体而言，该文适合作为内存解耦系统方向的重要参考，其"能力化隔离 + 原子字提交"的设计模式具备较强的可迁移性。

（完）
