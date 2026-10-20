---
title: "Teaching the Old Dog New Tricks: Building Efficient Data Pipelines for {Large-Scale} {LLM} {Pre-Training} (Operational Systems)"
title_zh: 老狗学新招：为大规模LLM预训练构建高效数据管道
authors: "Luofan Chen, Chenhan Wang, Weidong Zhang, Jinxin Chi, Hequan Zhang, Zanbo Wang, Chenyuan Wang, Lishu Luo, Sijin Wu, Junqi Hu, Jun Wang, Cheng Chen, Lixin Huang, Liyang Zhao, Yong Tian, Jun Guo, Youhui Bai, Wencong Xiao, Kang Chen, Cheng Li"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-chen-luofan.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 大规模LLM预训练的数据管道与跨数据中心流量瓶颈
tldr: 该论文针对大规模LLM预训练数据管道中的性能瓶颈，基于生产负载定量分析数据访问模式，揭示跨数据中心流量、检查点加载I/O竞争和多模态数据转换开销三类问题。作者据此提出相应的数据管道优化方法，以降低训练启动与评估延迟。结果表明这些优化能缓解数据加载瓶颈，对提升千卡级预训练作业效率具有实践意义。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: 大规模LLM预训练依赖数千GPU，数据管道效率直接影响作业性能，但生产负载中的数据访问瓶颈尚缺乏系统分析。
method: 论文定量分析生产数据访问模式，并针对跨数据中心流量、检查点I/O竞争和数据转换瓶颈设计数据管道优化。
result: 分析揭示三类此前低估的瓶颈，所提优化缓解了远程检查点评估与启动阶段的延迟和I/O竞争。
conclusion: 研究表明数据管道是LLM预训练性能的关键环节，针对数据访问模式的优化可提升大规模训练作业效率。
---

## Abstract
Data pipelines play a critical role in the performance of large-scale pre-training jobs running on thousands of GPUs. In this work, we present a comprehensive quantitative analysis of data access patterns from production workloads and reveal three previously underreported bottlenecks. First, cross-datacenter traffic emerges as a major source of latency when evaluating in-training models using remote checkpoints. Second, checkpoint loading during startup phases frequently suffers from I/O contention that delays job initialization. Third, data transformation during loading becomes a significant and CPU-intensive bottleneck for multimodal models. Guided by these findings, we introduce three optimizations: global-namespace-based predictive checkpoint replication, proactive hot-file replication, and offloading data transformation to storage-tier CPU resources. Crucially, we demonstrate that these optimizations are not system-specific but address fundamental architectural mismatches in the LLM era. They are broadly applicable to both legacy and modern storage systems, offering a high-return path to upgrade infrastructure with minimal engineering intrusion. Together, these techniques reduce wasted GPU hours per evaluation from 16,800 to 4,000, shorten checkpoint loading time at each training start by 40.8%, and reduce training stalls caused by dataloading by 63.2%.

---

## 论文详细总结（自动生成）

# 论文结构化总结：Teaching the Old Dog New Tricks（OSDI'26 Operational Systems）

## 1. 核心问题与整体含义

- **背景**：百亿至万亿参数基础模型的预训练需要数千至数万张 GPU 连续运行数周甚至数月。系统社区长期集中优化计算调度（Megatron、FSDP、ZeRO、nnScaler 等）与网络集合通信（NCCL、TCCL、AutoCCL 等）以提升 MFU，而负责加载、变换、投递 EB 级训练数据的**数据管道**却被严重忽视。
- **核心论点**：数据管道已不再是“被动工具”，而是预训练性能的主要决定因素之一。作者基于字节跳动生产环境（以 HDFS 为存储底座，承载 EB 级数据）中 90 天、3 万条训练作业 trace 的定量分析，指出通用存储系统在面对超大模型训练时存在三类此前被低报的瓶颈：
  1. **跨数据中心（cross-DC）延迟陷阱**：伴随式评估（companion evaluation）需从远端 DC 拉取 checkpoint，性能受限于数千次小张量读的 WAN RTT 而非带宽；
  2. **初始化 I/O 风暴**：作业重启时数千 rank 同步争抢少量热文件（全局 metadata、复制的 embedding 参数），产生拖尾读，阻塞整个集群；
  3. **多模态变换墙**：视频/图像解码等 CPU 密集变换无法被 GPU 计算掩盖，瓶颈从存储 I/O 转移到主机 CPU。
- **整体含义**：论文主张这些瓶颈的根源是“存储系统是反应式的，而训练负载本质是确定性的”这一架构错配；因此不必替换现有大数据存储栈，只需把应用层确定性暴露给存储层，即可用极小的工程侵入获得高回报。

## 2. 方法论

### 2.1 核心思想
利用训练负载的**可预测性/确定性**（评估周期固定、并行策略与 world size 在提交时已知、dataloader 全局顺序离线确定），把存储层从“被动响应”变为“主动预备”，实现“软件定义的垂直协同设计”。

### 2.2 三项关键技术

**(1) 预测式 Checkpoint 复制 + 信号驱动优先级（对应 §3 跨 DC 评估）**
- 伴随式评估通常每 ~1000 step 触发一次，周期可预测。训练端开始写 checkpoint 时即启动**流水线式批量复制**，把原本按张量粒度发起的上千次 WAN 读合并为大块连续传输，缓存到评估集群本地存储层。
- 轻量级服务端命名空间服务 **NNProxy** 将评估作业重定向到本地缓存分片，把 WAN 读转为本地读，同时消除 RTT 与小读碎片化。
- **信号驱动优先级**：对由 loss 尖峰/梯度异常触发的紧急评估，构造综合“模型规模 + 任务重要性 + 异常严重度”的优先级信号，存储调度器据此抢占后台流量（数据迁移、日志聚合等）带宽。

**(2) 主动热点预测 / 主动副本扩展（对应 §4 初始化 I/O 风暴）**
- 指出传统**反应式复制**（Scarlett 类）在启动场景失效：竞争窗口仅数十秒，检测+复制完成时风暴已过，且复制流量反而加剧拥塞。
- 提出协作接口 **`SetReplicationHints`**：训练框架在提交/写完 checkpoint 时告知存储层热点文件集合，分两类——
  - 全局 metadata：目标副本数与 world size N 成正比；
  - 复制型张量：依据 tensor→rank 映射聚合出每个文件 f 的期望并发度 ω_f。
- 副本数启发式公式：**R_target(f) = ⌈ ω_f / C_load ⌉**，其中 C_load 为单副本的安全并发负载能力。
- 临时副本带 **TTL**，启动阶段结束后自动回收，避免存储膨胀。
- 给出生产规范：**10k GPU 集群 64 副本，20k GPU 集群 128 副本**。

**(3) 存储侧变换卸载（对应 §5 多模态变换墙）**
- 前提：自研**确定性 dataloader**——20 GB 二进制 bin 文件 + 元数据映射表（约 1 万文件、0.7 TB）+ 每 step 的 info 文件（离线生成的全局顺序），实现全局随机 shuffle 而无需物理搬移数据，并支持精确字节区间访问（避免 Parquet 类列式格式的读放大）。
- 将存储层改造为**分离式预处理引擎**，三个机制：
  - **调度同步**：训练初始化时共享 dataset id 与 step 进度，存储节点自行解析后续数据块序列，省去客户端数百万次 RPC；
  - **JIT 变换**：存储节点维护 Consumer Queue，在 GPU 计算 Step N 时预读原始字节并执行解码/裁剪/归一化，为 Step N+1 备好张量；视频侧先做帧采样以约束回传网络载荷；
  - **负载感知回退**：节点 CPU 超过安全阈值（如 80%）即放弃变换、回传原始字节，客户端识别格式（Tensor vs Binary）后本地处理，形成带背压的混合模式。

### 2.3 被明确否决的替代方案（§6 “What If”）
- **P2P 加载**（NCCL/UCX）：非结构化 all-to-all 流量干扰同集群其他作业的延迟敏感集合通信；且 P2P 性能具概率性，最慢 peer 拖垮整个 gang 的同步屏障。故坚持分层 client-server 以获得确定性 SLA。
- **专用变换集群**（tf.data service / Ray Data 式）：解码后张量膨胀 50–100×，把瓶颈从 CPU 搬到网络；且需按多模态峰值配置、文本任务期闲置。
- **迁移到 AI 原生存储**（3FS、AIStore）：数据引力与运维连续性不可行（EB 级多业务数据湖、需全 NVMe/定制客户端、破坏 Spark 等生态兼容）。

## 3. 实验设计

- **数据来源 / 场景**：90 天采集的 3 万条生产训练 trace；从中选取 5 个代表性任务（占 GPU 总时长 70%）：
  - T-S（FSDP2，文本，CKPT 1–5 TB）、T-L（Megatron，文本，5–10 TB）、MM-S（FSDP，多模态）、MM-L（FSDP2，多模态）、MM-O（FSDP，多模态，5–10 TB）；集群规模约 4K–20K GPU。
  - 数据集规模示例：T-L 约 20 TB / 2 万文件；MM-L 约 6 PB / 超 300 万文件。
- **测量方法（benchmark 即生产负载本身，非标准数据集）**：
  - 存储层：HDFS client/server 端逐操作记录（open/read/pread/write/close、请求大小、时间戳、延迟）；
  - 训练层：PyTorch Profiler + 手工埋点（ckpt 加载/保存、dataloader fetch 与 transform、前反向、logits 保存、评估 merge/reshard）；
  - 按时间戳 join 两类记录，区分“原始存储服务时间”与“框架可见端到端延迟”。
- **对比方法**：主要是**改造前 vs 改造后**的 A/B 式生产对比，而非与其它系统横向对比：
  - 默认 **3 副本策略** vs 主动扩展至 **128 副本**（2048 GPU 受控实验，图 7 vs 图 9）；
  - 无预测复制 vs 有预测复制（跨 DC merge 延迟）；
  - 无存储侧卸载 vs 有卸载（P99 加载延迟、stall 时间、MFU、主机 CPU 占用）。
- **评估窗口**：跨 DC 部分为 30 天、19 个预训练任务、3589 次伴随式评估、捕获 156 次模型退化。

## 4. 资源与算力

- **明确给出的**：
  - 生产集群规模约 **4K–20K GPU**；受控实验为 **2048 GPU**；
  - 时间跨度：trace 采集 90 天，跨 DC 评估观测 30 天；
  - 间接算力指标：单次退化评估浪费 16,800 GPU 小时（优化后 4,000）；156 次退化累计浪费约 260 万 GPU 小时；优化共回收近 200 万 GPU 小时；多模态 straggler 每天浪费超 1 万 GPU 小时；启动延迟按集群推算年损失超 100 万 GPU 小时。
- **未明确说明的**：**GPU 型号/代际、互连规格（除提到评估集群用较旧、性价比高的硬件）、单作业训练总时长、存储节点数量与 CPU 型号、网络带宽具体配置（仅提到跨 DC 链路 60 Gbps 与 100 ms RTT）**。这削弱了结果的可复现性与横向可比性。

## 5. 实验数量与充分性

- **实验组数**：整体偏少，大致为
  - 5 条 trace 的 I/O 特征刻画（表 2）+ 文件大小分布（图 2）+ 张量大小 CDF（图 3）；
  - 跨 DC：吞吐/带宽争用测量（图 4、图 5）+ 单次评估阶段耗时分解（表 3）+ 30 天 3589 次评估的端到端收益；
  - 启动：启动延迟分解（表 4）、重启分布（图 6）、3 副本下的延迟/QPS（图 7）、下载延迟 CDF 与文件负载集中度（图 8）、128 副本对照（图 9）；
  - 多模态：加载延迟分解（表 5）、文本 vs 多模态最大延迟对比（图 11）、跨机 step 时间差（图 12）、卸载后 P99/stall/MFU/CPU 四项指标。
- **充分性评价**：
  - **优点**：规模真实（万卡级生产）、观测周期长（90 天/30 天）、指标多维（延迟分位数、QPS、GPU 小时、MFU、CPU 占用），并区分了均值与 P99/最大值，能反映拖尾问题。
  - **不足**：①**缺少与替代方案的实测对比**（P2P、专用变换集群、3FS 等只在讨论中定性否决，无数据）；②**几乎没有消融实验**——预测复制与信号优先级、JIT 与回退机制、调度同步各自的贡献未拆分；③受控实验仅 2048 GPU 一组，且绝对收益只有秒级（38.48 s → 22.78 s），外推到 20k 卡属推断；④未报告重复次数、方差或置信区间；⑤表 2 中部分单元以“–”缺失，MM-S 的读延迟 300.33 s 等异常值未解释；⑥副本数公式中的 C_load 未给出取值方法与敏感性分析；⑦80% CPU 回退阈值的选择依据未讨论。
  - **公平性**：作为 operational track 论文，以生产前后对比为主是可接受的，但对“收益归因”与“代价（额外副本存储、缓存一致性、TTL 回收开销）”缺少量化，客观性略打折扣。

## 6. 主要结论与发现

- **发现 1（跨 DC）**：单次多模态大模型评估可超 4 小时，I/O 占端到端约 **56.6%**，其中 **84.8%** 消耗在 checkpoint merging；MM-L checkpoint 中 **>60% 张量小于 16 KB**，小读在 100 ms RTT 下形成“锯齿状”吞吐，且 HDFS 128 MB 块导致约 **1.5× 读放大**；跨 DC 带宽被平均 208 个并发任务争抢，单任务可用带宽常低于 1 GB/s。
- **发现 2（启动）**：MM-S 启动总耗时 751 s，其中 HDFS 下载 300 s（**39.95%**）为最大瓶颈；open 延迟稳定，**read 才是尾延迟来源**，少数 straggler 读占总等待 **67.97%**；**前 5% 文件贡献 38.8% 的峰值 QPS**；根因是 checkpoint 系统的“并行写 + 去冗余”导致加载不对称——Embedding 等复制参数只存一份，恢复时形成一对多访问风暴，而 MoE 专家分片文件则天然分散。
- **发现 3（多模态）**：MM-L 数据加载中变换占 **94.4%**（5.05 s / 5.35 s），I/O 仅 13.6 ms；最慢主机单步 42.72 s vs 平均 5.35 s，单步跨机差异可达 5 s，日浪费超 1 万 GPU 小时；离线预变换不可行（视频体积膨胀 >40×，整体存储放大 50–100×，且对超参变化不灵活）。
- **量化收益**：
  - 跨 DC：merge 延迟平均降 **76.1%**（T-S 89.3%，MM-L 70.8%），单次退化浪费从 **16,800 → 4,000 GPU 小时**，观测窗口回收近 200 万 GPU 小时；
  - 启动：加载时间 **38.48 s → 22.78 s（−40.8%）**；
  - 多模态：P99 加载延迟 **−85.7%**，stall **−63.2%**，MFU 相对 **+10.8%**，训练主机加载 CPU 占用 **−94%**。
- **总体结论**：通过暴露应用层确定性，**遗留大数据系统
