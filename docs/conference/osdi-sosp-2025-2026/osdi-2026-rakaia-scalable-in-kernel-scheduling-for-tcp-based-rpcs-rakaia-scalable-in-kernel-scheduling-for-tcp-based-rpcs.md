---
title: "Rakaia: Scalable {In-Kernel} Scheduling for {TCP-Based} {RPCs}"
title_zh: Rakaia：面向基于TCP的RPC的可扩展内核内调度
authors: "Rui Yang, Konstantinos Prasopoulos, Edouard Bugnion"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-yang-rui.pdf"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 内核内RPC调度消除队头阻塞并降低延迟
tldr: 针对POSIX TCP按字节流暴露导致连接内外队头阻塞、gRPC等框架需在用户态重建消息语义并带来大量上下文切换与同步开销的问题，Rakaia将TCP抽象对用户态隐藏，提供纯消息接口并在内核内完成消息级调度。系统在多核上实现工作保持调度，消除队头阻塞并降低RPC时延，为高吞吐低延迟的数据中心RPC通信提供新路径。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: TCP按字节流暴露引发队头阻塞，RPC框架需在用户态重建消息语义，带来额外上下文切换与同步开销。
method: Rakaia对用户态隐藏TCP抽象，提供纯消息接口，并在内核内进行消息级、工作保持的调度。
result: 该设计在多核上消除连接内外队头阻塞，显著降低RPC时延与同步开销。
conclusion: 表明将消息语义下沉到内核可提升数据中心RPC的吞吐与低延迟表现。
---

## Abstract
Delivering RPCs with high throughput and low latency demands work-conserving scheduling across many CPU cores and eliminating head-of-line (HOL) blocking across all messages. By exposing per-connection byte streams rather than messages to userspace, the POSIX TCP API inherently induces HOL blocking both within and across connections. To mitigate HOL blocking, RPC frameworks such as gRPC must reconstruct message semantics in userspace through additional abstractions including dedicated I/O threads, work queues, and worker thread pools, introducing significant context switching and synchronization overheads. This paper presents Rakaia, a framework that hides all TCP-level abstractions from userspace and exposes a purely message-oriented API. By performing message parsing and work-conserving scheduling directly in the kernel’s TCP receive path, at the earliest possible point, Rakaia efficiently eliminates HOL blocking and avoids the heavy userspace machinery imposed by stream-based APIs. We implemented Rakaia as a Linux kernel module with support for kTLS. Rakaia is compatible with the kernel’s TCP stack and existing RPC protocols. We also adapted gRPC to use Rakaia’s API. Our evaluation shows Rakaia: (i) consistently eliminates HOL blocking across a wide range of connection counts; (ii) achieves up to 5× higher throughput-under-SLO than KCM, Linux’s current in-kernel message API over TCP; (iii) improves gRPC-Go’s throughput-under-SLO by up to 1.56×, and gRPC-C++’s by up to 2.69×; and (iv) improves the throughput-under-SLO for real-world applications including Silo running TPC-C and OpenTelemetry Collector by 1.39× and 1.42×, respectively.

---

## 论文详细总结（自动生成）

# Rakaia：面向基于 TCP 的 RPC 的可扩展内核内调度 —— 论文总结

## 1. 核心问题与研究背景

- **RPC 是数据中心通信的基石**：单个云应用可能由数百个微服务、数千个容器组成，全部通过 RPC 交互，因此尾延迟直接决定 SLO 达成率。
- **TCP 是事实标准，但流式抽象与消息型 RPC 天然错配**：POSIX TCP API 向用户态暴露的是**无消息边界的字节流**，导致两类队头阻塞（HOL blocking）：
  - **连接内 HOL**：同一连接上慢 RPC 阻塞后续 RPC；
  - **连接间 HOL**：连接间流量不均导致核心间负载失衡（如按 RSS 静态分连接）。
- **用户态框架的补偿代价高昂**：gRPC（基于 HTTP/2）必须在用户态用**专用 I/O 线程 + 工作队列 + 工作线程池**重建消息语义，带来大量上下文切换、跨线程通信与同步开销。实测把 gRPC-Go 服务器（20 硬件线程）的连接数从 80 提到 5,000，goroutine 数增长 16×，吞吐-under-SLO 反而下降 11%。
- **已有绕开 TCP 的方案难以落地**：Homa、eRPC、Shinjuku、DPDK 用户态栈、QUIC 等虽能消除 HOL，但需要定制客户端、非标准基础设施或专用硬件，在多租户云中部署困难。
- **KCM（Linux 现有内核消息 API）是唯一先例但设计有缺陷**：它在内核解析消息（正确方向），但**消息仍绑定于所属 TCP 连接、按连接局部调度**，因此仍有连接间 HOL；且 T 线程 × C 连接会产生 T×C 个 KCM socket 的二次方爆炸；不支持 kTLS。
- **核心问题**：能否在不引入复杂用户态机制、不牺牲可部署性的前提下，实现无 HOL 阻塞的高性能 RPC？
- **论文的整体含义**：提出 Rakaia，把消息解析与调度**下沉到内核 TCP 接收路径的最早点**，向用户态暴露**纯消息、连接无关**的 API，在保持 TCP/TLS 线缆兼容（客户端无需修改）的同时逼近理论上的消息共享（MS）排队模型。

## 2. 方法论

### 2.1 两个核心洞察

1. **在 softirq 中解析与调度**：内核本就在每个包到达时执行 TCP 接收路径，在此处切分消息并做调度决策的边际开销极小，可省掉整套用户态流水线。
2. **尽早将消息与连接解耦**：只有消息脱离连接归属，才能实现真正的全局（消息级）工作保持调度。

### 2.2 分析框架：三种排队模型（§2.2）

- **CP（连接分区 FIFO）**：连接静态分给线程，无同步开销，但同时受连接内/间 HOL 影响，非工作保持。
- **CS（连接共享 FIFO）**：连接放入共享池、空闲线程可 poll 任意连接，消除连接间 HOL，但处理期间连接仍是所有权单位，仍有连接内 HOL。
- **MS（消息共享 FIFO）**：显式抽取消息进共享队列，任意空闲 worker 取用，理论上同时消除两种 HOL，但需额外的抽取/入队阶段。
- 离散事件模拟结论：MS 尾延迟最优；CP 即使 5,000 连接也无法均衡；CS 随连接数增加而逼近 MS（CS-5,000 ≈ MS）。**推论：集中式 MS 的实现开销必须足够低才有意义**——这正是 Rakaia 的设计目标。

### 2.3 三大机制

- **① 每连接挂载消息解析器（扩展 strparser）**
  - 连接经 `rakaia_attach` 注册后，用 `rakaia_psock` 包装 TCP socket，并挂一个 strparser 实例。
  - 长度前缀协议（如 Memcached）：读定长头取长度，用二进制协议的 opaque 字段做请求/响应匹配。
  - HTTP/2（gRPC）：按帧头取 stream ID/length/type/flags，将完整帧追加到哈希表中的 per-stream 缓冲区；收到 `END_STREAM` 时合并为一条完整消息上送。用 `skb->cb` 保留空间维护 per-stream 链接，形成单条 skb 链。
  - **控制面下沉**：PING、WINDOW_UPDATE 等控制帧完全在内核处理（直接生成/应用窗口更新），无需系统调用与上下文切换。
- **② 逻辑集中、物理分布的调度层**
  - 每个 Rakaia socket 维护本地 FIFO；消息入队时用 **P2C（power-of-two-choice）** 随机采样两个队列、放入较短者，常数开销实现负载均衡，且**均衡效果与连接数无关**（少量连接也能保持队列长度有界）。
  - 若被选队列为空（说明有 worker 空闲），改为**直接交付（direct handoff）**，降低轻载延迟。
  - 本地队列耗尽时**工作窃取**：维护非空队列位图，再用 P2C 选受害者；全局无活可干才进入 idle。
  - 效果：轻载直投、重载靠窃取动态再平衡，逼近集中式队列而无其扩展性瓶颈。
- **③ 发送路径的轻量委托（delegation）**
  - 每条消息独立拷贝进新分配的 skb（不做同 socket 批量打包，避免内存压力下部分拷贝失败难以释放、以及额外延迟与同步复杂度）。
  - `recvmsg` 时缓存 `rakaia_psock` 指针，`sendmsg` 据此找到底层 TCP 连接，**不向用户态暴露连接信息**。
  - 不直接抢 TCP socket 锁（互斥锁会串行化并睡眠），而是先入 `virt_queue` 暂存，检查 `tx_in_use` 标志：已被占用则立即返回用户态；否则置位成为"指定发送者"，获取 socket 锁，把 `virt_queue` 全部搬入 `sk_write_queue`，循环复查并冲刷新到消息，直到队列为空才清标志、释放锁，最后调用 `tcp_push`。
  - 实现上绕过 `tcp_sendmsg`（其加锁/分配/拷贝逻辑已不需要），在 `rakaia_sendmsg` 中自行完成序号初始化、rx backlog 处理等，再调用内核既有 `tcp_push`。
- **API 设计**：新增协议族 `AF_RAKAIA` + `RAKAIAPROTO_CONNECTED`；每个工作线程仅需**一个** socket 承载所有 TCP 流量（epoll 只需监控一个 fd）；新增唯一控制操作 `rakaia_attach(rakaia_fd, csock, proto_id)`（ioctl）选择内置解析器；收发仍用 `recvmsg`/`sendmsg`。socket 不保留任何 per-connection 状态，仅是内核侧接口。
- **与 TLS 组合**：依赖 kTLS——握手在用户态完成后把会话密钥交给内核，kTLS 解密出的明文直接交给 strparser 解析；发送侧明文交给 kTLS 加密。因当前 kTLS 在 softirq 中并发解密不健壮且有竞争，Rakaia 暂用 **workqueue** 执行解密（作者计划打补丁支持安全的 softirq 解密）。
- **实现规模**：Linux v6.8 上约 **3,000 行 C 的可动态加载内核模块**，外加约 **60 行**注册新 socket 类型的补丁；不修改 TCP 栈语义。

## 3. 实验设计

- **硬件/环境**：CloudLab xl170 节点，单颗 Intel Xeon E5-2640v4（10 核 2.4 GHz，开启超线程 → **20 硬件线程**），Ubuntu 24.04 + Linux 6.8；服务器统一用 20 路多线程（gRPC-Go 用 20 个 worker goroutine）；**所有实验均为 compute-bound**。
- **负载生成**：扩展版 **Lancet**，先一次性建立全部连接，再按**泊松过程开环**发请求；指标为客户端观测的 **99 分位尾延迟 vs 达成吞吐（KQPS）**。
- **服务时间分布**（三种经典分布）：deterministic；exponential（均值 S̄）；bimodal（P[X=S̄/2]=0.9，P[X=5.5S̄]=0.1）。
- **场景维度**：任务粒度 100 μs 与 20 μs；连接数 20 / 80 / 5,000；TLS 开/关。
- **参考基准
