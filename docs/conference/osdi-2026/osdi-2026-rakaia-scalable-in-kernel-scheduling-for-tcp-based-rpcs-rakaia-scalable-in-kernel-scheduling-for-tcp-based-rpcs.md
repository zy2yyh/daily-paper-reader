---
title: "Rakaia: Scalable {In-Kernel} Scheduling for {TCP-Based} {RPCs}"
title_zh: Rakaia：面向TCP RPC的可扩展内核内调度
authors: "Rui Yang, Konstantinos Prasopoulos, Edouard Bugnion"
date: 2026
publication_date: 2026
publication_date_precision: year
publication_date_source: conference year only; exact release date unverified
publication_date_kind: unknown
pdf: "https://www.usenix.org/system/files/osdi26-yang-rui.pdf"
tags: ["query:dcn-comm"]
score: 5.0
evidence: 内核内消息调度消除队头阻塞以降低RPC时延
tldr: 高吞吐低时延的RPC需要跨多核的工作保持调度并消除消息级队头阻塞，但POSIX TCP按字节流暴露连接，天然导致连接内与连接间的阻塞。作者提出Rakaia，将消息语义与调度下沉到内核，向上暴露纯消息接口，避免用户态I/O线程与工作队列带来的上下文切换开销。实验表明该设计在多核上实现了高吞吐与低时延的RPC服务。工作为数据中心通信栈的消息化与调度协同提供了新方案。
source: OSDI-2026-USENIX
selection_source: conference_retrieval
motivation: TCP按字节流暴露连接导致消息级队头阻塞，迫使RPC框架在用户态做额外调度并付出切换开销。
method: Rakaia将消息语义与调度下沉到内核，向用户态暴露纯消息接口并跨多核做工作保持调度。
result: 该框架消除了连接内与连接间的队头阻塞，在多核上取得高吞吐与低时延的RPC性能。
conclusion: 说明将消息化与调度协同下沉内核可显著改善数据中心RPC通信效率。
---

## Abstract
Delivering RPCs with high throughput and low latency demands work-conserving scheduling across many CPU cores and eliminating head-of-line (HOL) blocking across all messages. By exposing per-connection byte streams rather than messages to userspace, the POSIX TCP API inherently induces HOL blocking both within and across connections. To mitigate HOL blocking, RPC frameworks such as gRPC must reconstruct message semantics in userspace through additional abstractions including dedicated I/O threads, work queues, and worker thread pools, introducing significant context switching and synchronization overheads. This paper presents Rakaia, a framework that hides all TCP-level abstractions from userspace and exposes a purely message-oriented API. By performing message parsing and work-conserving scheduling directly in the kernel’s TCP receive path, at the earliest possible point, Rakaia efficiently eliminates HOL blocking and avoids the heavy userspace machinery imposed by stream-based APIs. We implemented Rakaia as a Linux kernel module with support for kTLS. Rakaia is compatible with the kernel’s TCP stack and existing RPC protocols. We also adapted gRPC to use Rakaia’s API. Our evaluation shows Rakaia: (i) consistently eliminates HOL blocking across a wide range of connection counts; (ii) achieves up to 5× higher throughput-under-SLO than KCM, Linux’s current in-kernel message API over TCP; (iii) improves gRPC-Go’s throughput-under-SLO by up to 1.56×, and gRPC-C++’s by up to 2.69×; and (iv) improves the throughput-under-SLO for real-world applications including Silo running TPC-C and OpenTelemetry Collector by 1.39× and 1.42×, respectively.

---

## 论文详细总结（自动生成）

# Rakaia：面向 TCP RPC 的可扩展内核内调度 — 论文总结

## 1. 核心问题与研究背景

- **背景**：RPC 是数据中心几乎所有通信的基础，单一云应用可能由数百微服务、数千容器通过 RPC 交互，核心目标是压低尾延迟以满足严格 SLO。TCP 仍是生产环境事实标准的传输协议（配合 TLS 提供安全）。
- **根本矛盾**：TCP 是面向字节流的连接抽象，而 RPC 是面向消息的。POSIX TCP API 向用户态暴露无消息边界的字节流，天然造成两类队头（HOL）阻塞：
  - **连接内 HOL 阻塞**：同一 TCP 流上慢 RPC 阻塞后续 RPC；
  - **连接间 HOL 阻塞**：请求在连接间分布不均，导致跨核负载失衡。
- **现有用户态方案的代价**：gRPC 等框架必须在用户态重建消息语义——专用 I/O 线程解复用字节流、工作队列传递消息、worker 线程池执行 RPC，带来大量上下文切换、跨线程通信与同步开销。文中给出实例：gRPC-Go 服务器连接数从 80 增至 5,000，goroutine 数增长 16×，吞吐-under-SLO 反而下降 11%。
- **已有替代路径的局限**：
  - 放弃 TCP 的方案（Homa、eRPC、Shinjuku、QUIC 等）虽能根治 HOL，但需自定义客户端、非标准基础设施或专用硬件，难以在多租户云中部署；
  - KCM 是 Linux 首个内核态消息 API 尝试，虽在内核解析消息，但消息仍绑定连接、调度是连接局部的，存在连接间 HOL 阻塞，且 T×C 个 KCM socket 呈二次方膨胀，不支持 kTLS。
- **核心问题**：能否在不引入复杂用户态机制、不牺牲可部署性的前提下，实现无 HOL 阻塞的高性能 RPC？

## 2. 方法论

### 2.1 两个核心洞察
- **洞察一**：消息解析与调度应放在处理入站 TCP 包的**同一 softirq 上下文**中——内核本就在每个包到达时执行 TCP 接收路径，此处附加消息提取与调度开销极小，且无需另建用户态流水线。
- **洞察二**：消息必须**与底层 TCP 连接解耦**，才能实现全局（跨连接）调度与真正的工作保持（work-conserving）。

### 2.2 三大机制
- **每连接消息解析器**：基于 `strparser` 在内核从 TCP 字节流直接提取离散消息，避免应用层解复用。
- **逻辑集中、消息粒度的请求队列**：将消息调度与连接解耦，同时消除连接内与连接间 HOL 阻塞。
- **发送路径轻量委托机制**：避免多线程经同一 TCP socket 发送时的锁争用。

### 2.3 系统架构（三层）
- **与 TCP/IP 栈的钩子**：不改动 TCP/IP 栈，复用 `strparser` 并从仅支持长度定界扩展到支持 HTTP/2；每 TCP socket 附加协议相关解析器，从接收队列增量重组完整消息。
- **调度层**：逻辑集中、物理分布——每核一个 FIFO 队列；新消息用 **power-of-two-choice (P2C)** 采样两个队列放入较短者；worker 优先取本地队列，空则**工作窃取**（用位图跟踪非空队列 + P2C 选受害者）；轻载时**直接交接**（direct handoff），重载时队列自然堆积并动态再平衡，逼近集中式队列而无其扩展性瓶颈。
- **消息收发**：每应用线程一个面向消息的 socket，接收时从调度器交付完整消息，处理后经同一 socket 沿关联 TCP 连接回传。

### 2.4 API 设计
- 新协议族 `AF_RAKAIA` + `RAKAIAPROTO_CONNECTED`，每 worker 线程仅需一个 socket（epoll 只需监控一个 fd）。
- 唯一新控制操作 `rakaia_attach(rakaia_fd, csock, proto_id)`：将已有 TCP 连接关联到 Rakaia 并选择内置解析器（Memcached、HTTP/2/gRPC），之后用标准 `recvmsg`/`sendmsg` 收发完整消息。
- socket 不维护任何 per-connection 状态，传输状态全留在 TCP 栈中。

### 2.5 接收路径关键细节
- **连接注册**：`rakaia_attach` 时创建 `rakaia_psock` 包装目标 TCP socket，持有 `strparser` 实例及发送路径所需字段。
- **协议解析**：
  - Memcached：检查定长头提取长度，用二进制协议 opaque 字段匹配请求/响应；
  - HTTP/2：解析帧头（stream ID、length、type、flags），按 stream 在哈希表中缓冲帧，遇 `END_STREAM` 合并为完整消息；PING、WINDOW_UPDATE 等控制帧完全在内核处理，可即时流控与应答。
- **来源跟踪**：消息首个 skb 内嵌 `rakaia_psock` 指针，`recvmsg` 拷到用户态时提取缓存，供后续 `sendmsg` 定位正确 TCP 连接而不向用户态暴露连接信息。

### 2.6 发送路径关键细节
- **独立消息复制**：每条消息独立复制进新分配的 skb；不采用同 socket 批量打包，以规避内存压力下部分复制失败难以释放含多消息 skb 的边角问题。
- **两阶段委托传输**（Algorithm 1）：
  1. 消息先入 `rakaia_psock` 的虚拟发送队列 `virt_queue`（不触碰 TCP socket 锁）；
  2. 检查 `tx_in_use` 标志：若已被占用则直接返回用户态（委托路径）；
  3. 否则置位成为指定发送者，获取 TCP socket 锁，将 `virt_queue` 全部移入 `sk_write_queue`，处理 rx backlog，调用 `tcp_push` 发出；
  4. 循环重查 `virt_queue` 直至为空，清标志并释放锁，避免遗漏并发入队消息。

### 2.7 与 TLS 的组合
- 依赖内核 TLS（kTLS）：握手在用户态完成，会话密钥交给内核，记录处理卸载到 kTLS。
- 接收路径：kTLS 拦截密文解析为 TLS 记录，交给附加的 `strparser`，后者向 kTLS 解密模块取明文再解析应用消息；发送路径：明文交 kTLS 加密模块后经 TCP 发出。
- 当前 kTLS 在 softirq 中并发解密不安全且有争用，Rakaia 暂用 workqueue 执行解密，并计划打补丁支持可靠的 softirq 解密。

### 2.8 实现要点
- Linux v6.8 上约 **3,000 行 C 代码**的可动态加载内核模块，另加约 **60 行内核补丁**注册新 socket 类型，不改 TCP 栈语义。
- 扩展 `strparser` 支持 HTTP/2 多帧重组：保留帧级 `frag_list` 链接，用 `skb->cb` 保留空间维护 per-stream 链接，形成每 stream 单一 skb 链。
- 卸载 HTTP/2 传输逻辑（流控、PING 等）到内核，省去系统调用与上下文切换。
- 自定义发送路径直到 `tcp_push`：因无需 `tcp_sendmsg` 中的锁获取、skb 分配与数据复制，改为在 `rakaia_sendmsg` 中直接做序列号初始化、rx backlog 处理，再调用内核既有 `tcp_push`。

## 3. 实验设计

### 3.1 平台与负载
- **硬件**：CloudLab xl170 节点，单颗 Intel Xeon E5-2640v
