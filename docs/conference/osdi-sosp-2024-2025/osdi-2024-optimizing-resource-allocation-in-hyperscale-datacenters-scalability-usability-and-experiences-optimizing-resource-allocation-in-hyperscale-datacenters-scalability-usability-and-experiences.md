---
title: "Optimizing Resource Allocation in Hyperscale Datacenters: Scalability, Usability, and Experiences"
title_zh: 超大规模数据中心资源分配的优化：可扩展性、易用性与实践经验
authors: "Neeraj Kumar, Pol Mauri Ruiz, Vijay Menon, Igor Kabiljo, Mayank Pundir, Andrew Newell, Daniel Lee, Liyuan Wang, Chunqiang Tang"
date: 2024
publication_date: 2024
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi24-kumar.pdf"
tags: ["query:dcn-comm"]
score: 4.0
evidence: 面向超大规模数据中心的资源分配，涵盖流量路由等优化问题
tldr: 论文面向Meta拥有数百万台服务器的私有云环境，指出资源分配涉及硬件放置、服务器分配、训练与推理部署、流量路由等诸多优化问题。实际落地的主要障碍是策略难以写成精确数学形式，以及问题规模带来的可扩展性挑战。作者提出可复用的资源分配框架，兼顾易用性与大规模求解效率，并总结了生产部署经验。其价值在于为超大规模数据中心的资源与流量调度提供工程化方法，但与集合通信算法本身关联有限。
source: OSDI-2024-USENIX
selection_source: conference_retrieval
figures_json: "[{\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-optimizing-resource-allocation-in-hyperscale-datacenters-scalability-usability-and-experiences/fig-001.webp\", \"caption\": \"\", \"page\": 1, \"index\": 1, \"width\": 875, \"height\": 356}, {\"url\": \"assets/figures/osdi-2024-usenix/osdi-2024-optimizing-resource-allocation-in-hyperscale-datacenters-scalability-usability-and-experiences/fig-002.webp\", \"caption\": \"\", \"page\": 1, \"index\": 2, \"width\": 875, \"height\": 360}]"
motivation: 超大规模数据中心资源分配问题多且规模大，策略建模难、求解可扩展性差。
method: 构建可复用的资源分配优化框架，兼顾策略表达的易用性与大规模问题求解效率。
result: 在Meta生产环境中支撑了硬件放置、任务部署与流量路由等多类优化。
conclusion: 为超大规模数据中心资源与流量调度提供了可落地的工程化框架。
---

## Abstract
Meta's private cloud uses millions of servers to host tens of thousands of services that power multiple products for billions of users. This complex environment has various optimization problems involving resource allocation, including hardware placement, server allocation, ML training & inference placement, traffic routing, database & container migration for load balancing, grouping serverless functions for locality, etc. The main challenges for a reusable resource-allocation framework are its usability and scalability. Usability is impeded by practitioners struggling to translate real-life policies into precise mathematical formulas required by formal optimization methods, while scalability is hampered by NP-hard problems that cannot be solved efficiently by commercial solvers. These challenges are addressed by Rebalancer, Meta's resource-allocation framework. It has been applied to dozens of large-scale use cases over the past seven years, demonstrating its usability, scalability, and generality. At the core of Rebalancer is an expression graph that enables its optimization algorithm to run more efficiently than past algorithms. Moreover, Rebalancer offers a high-level specification language to lower the barrier for adoption by systems practitioners.

---

## 论文详细总结（自动生成）

# 《Optimizing Resource Allocation in Hyperscale Datacenters》论文总结

## 一、核心问题与整体含义

- **背景**：Meta 私有云拥有数百万台服务器，承载数万个服务、支撑数十亿用户。日常运营中涌现出大量资源分配问题：硬件（机架）放置、服务放置（配额/预约）、服务分片迁移、流量路由、Serverless 函数局部性分组、数据库/容器迁移、ML 训练与推理部署等。
- **共同模式**：把一组对象（objects）分配到一组箱子（bins），在满足约束的同时优化目标。这类问题天然可写成混合整数规划（MIP），以二值变量 `v_ij` 表示对象 i 是否分配到箱子 j。
- **两大核心障碍**：
  - **易用性**：系统工程师难以把现实中的复杂、甚至临时性的策略翻译成 MIP 所需的精确数学公式。已有工作（DCM 用 SQL、Wrasse 用 DSL）虽有抽象，但缺乏大规模生产验证，且未解决可扩展性。
  - **可扩展性**：MIP 模型输入规模为 O(|O|×|B|)。Meta 的问题可达数百万对象、10 万箱子，即 O(10^11) 个变量；且分配问题普遍 NP-hard，商业求解器无法应对。历史上 Google Slicer 用手工启发式，Azure Service Fabric 尝试 MIP 失败后转向模拟退火，Meta 自家 Shard Manager 的手工启发式也因复杂度过高难以为继。
- **整体含义**：论文提出 **Rebalancer**——一个通用分配问题求解框架，历经 7 年、覆盖数十个大规模生产用例，证明其易用性、可扩展性与通用性。核心是一个**表达式图（expression graph）**，既是紧凑的模型表示，也是高效局部搜索算法的基础，并配套高层声明式规格语言。

## 二、方法论

### 2.1 总体架构：三个核心问题
设计求解器需解决**模型规格（model specification）**、**模型表示（model representation）**、**模型求解（model solving）**三件事。

### 2.2 模型规格：三层抽象
- **建模构件**：
  - *维度（Dimension）*：对象/箱子到数值的映射（如 CPU、内存容量与需求，也可编码"禁止分配"等复杂关系）。
  - *箱子层级（Bin hierarchy）*：用 scope 表示服务器、机架、数据中心等层级，scope item 是箱子集合。
  - *对象层级（Object hierarchy）*：用 partition/group 聚合对象（如同一 job 的 tasks、同一分片的副本）。
- **利用率（utilization）**：核心概念，定义为
  `util(b_j, D) = Σ_{o_i∈O} D(o_i) · v_ij`
  并推广到 scope item 与对象组（如 `util(rack r, job j, ObjectCount)`）。
- **利用率的多种"时态"**：为刻画迁移过渡期双占用、系统稳定性等场景，引入 AFTER、INITIAL、STAYED 以及派生量 NEW = AFTER−STAYED、OLD = INITIAL−STAYED、ANY = INITIAL+AFTER−STAYED（避免重复计数）。实现上为每个箱子引入维度 `Dinit_j`，把 STAYED 归约为一次 AFTER 形式的计算。
- **表达式 API**：提供 Max、Sum 等聚合算子与 Step、Ceil、Log、Power 等变换算子，可表达非线性，几十行到几百行代码即可实现新 spec。
- **高层 spec API**：封装数十个常用目标与约束（CapacitySpec、GroupCountSpec、BalanceSpec、MinimizeMovementSpec、ColocateGroupsSpec、DrainCapacitySpec 等，论文列出 28 个中的 21 个常用项及其使用次数与代码行数）。
- **实践数据**：平均每个问题使用 7 个 spec（最多 14 个）；85% 的约束与目标直接复用已有 spec，无需写表达式 API。

### 2.3 模型表示：表达式图
- 每个 spec 被翻译为表达式的递归组合，所有约束/目标汇入一个有向无环图 G（节点为表达式，边为输入）。
- **规模压缩的关键——Object Lookup**：多数情况下对象的维度值不随目标箱子变化（如任务内存需求与所在服务器无关），因此为每个静态维度建立**对象向量 V_D**（大小 O(|O|)），用 `util(S_i, D) = Lookup(S_i, V_D)` 表示对 scope item 内箱子的聚合。Lookup 本身只保存引用（常数大小），V_D 与 scope 表示被所有表达式共享复用，使整体模型规模从 Θ(|O|×|B|) 降为 **Θ(|O|+|B|)**。
- 约束形如 `f(·) ≤ 0`，可用 Max 合并：例如 CapacitySpec 的 |B| 条约束可写成 `Max_i(Lookup(b_i,V_D) − L_i) ≤ 0`。

### 2.4 模型求解
- **小问题走 MIP**：每个表达式实现递归的 `mipTranslate`，转成二值变量的线性组合，交给 Gurobi/Xpress 等商业求解器；利用等价对象类合并变量（从 O(|O|·|B|) 降到 O(|O_d|·|B_d|)），并支持只把"热箱子"子集设为动态箱子做部分求解。
- **大问题走图辅助局部搜索**（Algorithm 1）：循环执行 ①生成候选移动 ②评估 ③应用最优移动，直到无改进。与既有局部搜索的本质区别在于三步都借助表达式图完成。
- **候选生成**：候选移动总数为 |O|×(|B|−1)，故按**箱子**而非对象限制搜索空间（箱子数通常远少于对象数）。核心是**热箱子（hottest bin）**选择：按叶节点对目标的贡献贪心排序，用增量优先队列处理多箱子并列，前序遍历表达式图、按节点潜力降序展开，一旦队列出现唯一最优即提前退出；可在动态/静态排序间权衡。
- **移动策略**：SINGLE（穷举源箱子内每个对象到每个目标箱子）、SINGLE_GREEDY（接受首个改进）、SINGLE_RANDOM（目标箱子随机采样，常用 f=10%、上限 1k 箱子）、SWAP（交换）、KL_SEARCH（借鉴 Kernighan–Lin），以及利用领域知识的自定义策略（如 Shard Manager 优先评估大分片）。
- **高效评估**：只重算受影响的节点——预处理"对象→叶节点""箱子→叶节点"映射，自底向上传播变更；节点重算只遍历变化的子节点（如 Max 节点维护按值排序的子节点表，取变化子节点的最大值 z1 与首个未变化子节点的值 z2，新值为 max(z1,z2)，代价 O(c log ℓ) 而非 O(ℓ)）。评估操作不改图状态，故可**多线程并行**，多数大实例可达约 15 万次评估/秒（最高约 50 万次/秒）。
- **等价对象识别**：若交换两对象的箱子不改变任何约束与目标表达式，则二者等价；用基于表达式图的贪心递归切分算法求出等价类，搜索时每类只探索一个对象。
- **其他工程细节**（附录）：增量更新的数值稳定性（Sum 节点用线段树避免精度累积）、全局/局部最优性退出条件、约束下界估计等。

### 2.5 案例研究（规格层面）
- **硬件放置**：机架为对象、(周, 位置) 为箱子；用 CapacitySpec 建模电力/网络，AiRack 维度约束 AI 机架进入 AI 区，ColocateGroupSpec 约束同周放置；规模数百对象、数千箱子，用 MIP 求解。
- **服务放置**：服务器为对象、reservation 为箱子；CapacitySpec 满足容量、GroupCountSpec 保证跨 MSB/机架散布、MinimizeMovementSpec 控制抖动、SingleGroupFailureBufferSpec 保证单 MSB 故障冗余；最大 70 万服务器 × 6k reservation，每小时每区域求解一次，10 分钟 SLO，已从 MIP 切换到局部搜索。
- **服务分片**：分片为对象、Linux 进程为箱子；用 ANY util 处理双占用，用 NEW/OLD util 限制抖动，AssignmentAffinitiesSpec 表达区域偏好，BalanceSpec 做区域/全局负载均衡；最大 180 万对象 × 2.7 万箱子，5 分钟时限。
- **消息队列放置**：用户聚合为 bundle 作为对象、区域为箱子；AssignmentAffinitiesSpec 最小化时延，ColocateGroupSpec 共置，DrainCapacitySpec 保证单区域故障时的溢出容量；规模小、每周求解一次，用 MIP。
- **Kubernetes 调度器**：为与 DCM 对比，用现有 spec 复现 Kubernetes 调度策略（约 500 行代码，对比 DCM 的 550 行 SQL）。
- **其他**：容器再平衡、边缘到主数据中心流量路由、Serverless 函数分组、ML 训练/推理跨区域部署、工单分配等。

## 三、实验设计

- **数据集/场景**：
  - **生产流量统计**（图 3）：一周内数千万次求解，覆盖 §3 各类用例；最大为服务分片（180 万分片、2.7 万服务器）与服务放置（70 万服务器、6k reservation）；P99 求解时间 16 秒（6.5 万对象、5k 箱子规模下）。
  - **Azure 公开数据集**（用于 Kubernetes 调度对比 DCM）。
  - **自构造病态用例**：参数 N，3N 个 pod、5N 个节点，内存需求按 5 组指数分布，唯一最优解要求每节点恰好用满 32GB。
  - **生产服务放置数据**（与 POP 式分区 MIP 对比）。
  - **生产服务分片数据**（表达式图内存随规模增长验证）。
- **Benchmark 与对比方法**：
  - **DCM**（SQL + OR-tools CP-SAT）：分 DCM 规模（每批 50 pod，1k/5k/10k 节点）与超大规模（每批 500–5k pod，10k/50k/100k 节点）；对比 SINGLE_GREEDY 与 SINGLE_RANDOM（f=10%，上限 1k 箱子）。
  - **最优 MIP 求解器**：在 500 节点上放 1 万个 pod，比较放置率与延迟。
  - **POP 式分区 MIP**：在 4 个服务放置实例上比较运行时间与目标值差距。
  - **局部搜索内部消融**：表达式图内存线性增长（7.1 万/15.2 万/28.9 万对象，1.7/3.2/6.2 GB）、
