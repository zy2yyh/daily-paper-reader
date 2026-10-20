# 日报 · 2026-09-21 ~ 2026-10-20

- 最近生成时间：2026-10-20 11:29:39 UTC
- 今日累计更新：1 次
- 今日累计推荐总数：19
- 精读区：12
- 速读区：7

## 今日简报（AI）
本期（9/21–10/20）共读19篇（精读12、速读7），焦点集中在分布式训练通信与集合通信优化。最值得看两篇满分工作：NCCL M2N用布局与拓扑感知优化张量重分片集合通信，DualColl用分数树打包逼近带宽受限集合通信的吞吐最优。普通读者可先读这两篇的动机与评测，再结合速读中的AutoLoCo与MoE专家路由，理解通信节省如何落地。

## 精读区
1. [NCCL M2N: A Layout- and Topology-Aware Collective for Distributed Tensor Resharding](/20260921-20261020/2610.07516v1-nccl-m2n-a-layout--and-topology-aware-collective-for-distributed-tensor-resharding) （10.0/10）
2. [DualColl: Throughput-Optimal Bandwidth-Bound Collective Communications via Fractional Tree Packing](/20260921-20261020/2610.15704v1-dualcoll-throughput-optimal-bandwidth-bound-collective-communications-via-fractional-tree-packing) （10.0/10）
3. [SP-Allreduce: Symmetric Tree Schedules for Tori and Beyond](/20260921-20261020/2610.16387v1-sp-allreduce-symmetric-tree-schedules-for-tori-and-beyond) （10.0/10）
4. [HOCCL: Offloading Collective Communication from GPU Cores to Accelerate Distributed Training](/20260921-20261020/2609.34334v1-hoccl-offloading-collective-communication-from-gpu-cores-to-accelerate-distributed-training) （9.0/10）
5. [VarioPath: Workload-Aware All-to-All Communication for PCIe GPU Clusters](/20260921-20261020/2609.34340v1-variopath-workload-aware-all-to-all-communication-for-pcie-gpu-clusters) （9.0/10）
6. [Purlin: Separating Orchestration from the Datapath of Collectives](/20260921-20261020/2609.36954v1-purlin-separating-orchestration-from-the-datapath-of-collectives) （9.0/10）
7. [Joint Effects of GPU Server Topology, Parallelism, and Congestion Control on MoE Inference: A Controlled Simulation Study](/20260921-20261020/2609.37828v1-joint-effects-of-gpu-server-topology-parallelism-and-congestion-control-on-moe-inference-a-controlled-simulation-study) （9.0/10）
8. [T-CCL: Resource Efficient and Performant Collective Communication using Tensor Memory Accelerator](/20260921-20261020/2610.07098v1-t-ccl-resource-efficient-and-performant-collective-communication-using-tensor-memory-accelerator) （9.0/10）
9. [Ofan: Optimal Load Balancing for AI Training](/20260921-20261020/2610.07230v1-ofan-optimal-load-balancing-for-ai-training) （9.0/10）
10. [Deploying RDMA for Distributed Inference and Training At Scale](/20260921-20261020/2610.12815v2-deploying-rdma-for-distributed-inference-and-training-at-scale) （9.0/10）
11. [PCM: Programmable Congestion Management for Terabit RDMA NICs](/20260921-20261020/2610.13130v1-pcm-programmable-congestion-management-for-terabit-rdma-nics) （9.0/10）
12. [MCCL: Measurement-based Collective Communication Library](/20260921-20261020/2610.17091v1-mccl-measurement-based-collective-communication-library) （9.0/10）

## 速读区
1. [Balancing Generality and Specialization: A Survey on AI Datacenter Hardware Architecture](/20260921-20261020/2609.26829v1-balancing-generality-and-specialization-a-survey-on-ai-datacenter-hardware-architecture) （8.0/10）
2. [AutoLoCo: Communication Efficient Distributed LLM Training via Adaptive Synchronization](/20260921-20261020/2609.36662v1-autoloco-communication-efficient-distributed-llm-training-via-adaptive-synchronization) （8.0/10）
3. [Memory-Efficient Expert Routing for Distributed MoE Training](/20260921-20261020/2610.07333v1-memory-efficient-expert-routing-for-distributed-moe-training) （8.0/10）
4. [Accelerating MoE Training through Expert Placement and Replica Coordination](/20260921-20261020/2610.14798v1-accelerating-moe-training-through-expert-placement-and-replica-coordination) （8.0/10）
5. [A New Lens on Multi-GPU Communication: API-Level Memory Models for Portable Optimization](/20260921-20261020/2610.16190v1-a-new-lens-on-multi-gpu-communication-api-level-memory-models-for-portable-optimization) （8.0/10）
6. [TopoEP: Topology-Aware Load Balancing for Expert-Parallel MoE Training](/20260921-20261020/2609.35481v1-topoep-topology-aware-load-balancing-for-expert-parallel-moe-training) （7.0/10）
7. [Zepp: Accelerating Distributed MoE Serving under Relaxed Balance Constraints](/20260921-20261020/2610.11158v2-zepp-accelerating-distributed-moe-serving-under-relaxed-balance-constraints) （7.0/10）

---
使用键盘方向键可在日报/论文之间快速切换。
