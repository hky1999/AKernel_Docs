# AKernel、Agent Substrate 与 AgentENV 源码级对比及服务器性能评测可行性调研

> 调研时间：2026-08-01 17:10:09（UTC+8）  
> 文档时间戳：20260801T091009Z  
> AKernel 论文版本：b6e0d2ec286271ee7742d3667b607f1a4f04f219  
> AKernel 源码版本：5684c858cd835db053adef1265419af64e67bd04  
> distill-fs 子模块版本：5f3d6d3979a4ed76d4bb6722d7c1ce6db7126634  
> sandboxd 子模块版本：60e6a33e1f2b1ae04c10586c238f6d0200d0c315  
> Agent Substrate 版本：cbdeb7dbe003a55a16960a301bc595d9aa38b1ad  
> AgentENV 版本：6296bc4be7ad79eb3a278eb5264ef011c341adf5  
> 调研方法：论文只用于确认 AKernel 的研究目标；README、部署清单、API 和运行时源码用于确认当前已实现能力。本次没有实际创建云资源，也没有在真实服务器上执行三套系统的 benchmark，因此本文给出的是可复现的部署与实验方案，不把设计目标或 README 演示数字当成实测结果。

## 1. 结论摘要

### 1.1 三者不是完全同构的产品

三套系统都在处理 Agent 执行环境，但它们目前最强的层次不同：

| 系统 | 当前最准确的定位 | 当前最有辨识度的能力 |
|---|---|---|
| AKernel 论文 | 面向 Agent 的可编程数据中心内核 | AProc、ARealm、APlane、ASyscall；把执行、状态、策略和集群调度统一成 cloud-OS 语义 |
| AKernel 当前源码 | openYuanRong 控制面 + gVisor sandboxd + distill-fs lazy rootfs + Python SDK/部署系统 | 集群级 sandbox 创建和交互、per-sandbox 资源请求、gVisor 隔离、按需 rootfs、chunk cache/dedup/P2P |
| Agent Substrate | Kubernetes 上的有状态 Actor/Worker 多路复用控制面 | 大量逻辑 Actor 映射到少量预热 Worker、显式 suspend/resume、location-transparent routing、request parking |
| AgentENV | Firecracker microVM sandbox 服务和深度快照数据面 | E2B API、OverlayBD/ublk rootfs 与增量内存快照、lazy restore、共享 memory ublk/page cache、运行态 balloon free-page reporting |

因此不能只做一张“启动时间排行榜”。至少要把实验拆成：

1. 三者共同支持的端到端环境创建与第一条有效动作；
2. AKernel gVisor 与 Substrate gVisor 的同 runtime 对照；
3. Substrate Cloud Hypervisor 与 AgentENV Firecracker 的 microVM 系统对照；
4. 仅 Substrate 与 AgentENV 参加的 suspend/resume、snapshot 和 idle scale-to-zero 对照；
5. AKernel 将论文中的 checkpoint-on-idle 补齐以后，再加入三者完整状态恢复实验。

### 1.2 能不能找服务器部署起来做性能比较

答案是：**可以，而且值得做；但应分阶段部署，并明确后端差异。**

- AKernel 当前有单机 privileged all-in-one container + Traefik 的 standalone 路径，也有 Helm 和 Terraform 资产。
- Agent Substrate 有可直接执行的 kind quickstart。gVisor 路径不需要 KVM；microVM 路径可在带 /dev/kvm 的 Linux 服务器上运行，但还要组装并分发 Kata、Cloud Hypervisor、virtiofsd、guest kernel 和 rootfs 资产。
- AgentENV 有安装脚本、Docker、Docker Compose 和 Kubernetes 部署路径。单节点 Docker/系统服务是三者中验证 Firecracker 路径最直接的方式。

严格 A/B 测试应在同一物理服务器上**顺序运行**，不能让三套系统同时运行。它们会争用 CPU/NUMA、KVM、page cache、NVMe、registry/object-store 带宽、网络 namespace 和 iptables，导致结果不可解释。

同一默认 OS 镜像并不一定能无摩擦满足三者：

- AKernel v0.1.0 明确要求 Linux x86-64 + cgroup v1；
- AgentENV 要求 Linux 6.8+、/dev/kvm 和 ublk，官方安装脚本额外要求 Ubuntu 24.04；Ubuntu 24.04 默认通常是 cgroup v2；
- Substrate 依赖当前 Kubernetes、Docker/kind 或 GKE，其 gVisor 与 microVM worker 还各有不同权限和资产要求。

最稳妥的实验基础设施是以下二选一：

1. 同一台裸金属服务器使用不同启动盘/系统镜像，逐系统重装或切换后重复测试；
2. 两台或更多同型号服务器做 cross-over：每套系统都在每台服务器上跑一轮，用来消除单机体质和盘况偏差。

### 1.3 AKernel 论文愿景与当前实现必须分开

AKernel 论文的中心想法很强：Agent 看到的是跨节点、可恢复、带资源和策略语义的数据中心进程，而不是 Pod 或某个具体 VMM。但当前论文仍是设计稿：

- abstract 为空；
- implementation 章节是计划性说明；
- evaluation 章节只列实验计划；
- workload 数据、调度代价模型、checkpoint 语义、runtime selection 和 policy/provenance 均有待补 TODO；
- 没有报告任何 p50/p95/p99、吞吐、成本、资源节省或任务正确率数字。

当前 AKernel 源码已经是一套可部署的 gVisor sandbox substrate，但尚没有公开的 sandbox checkpoint、restore、fork、pause、resume 或 hibernate API。因此：

- 论文中的 AProc/ARealm/APlane/ASyscall 是**目标抽象**；
- 当前源码中的 Sandbox/openYuanRong/sandboxd/distill-fs 是**已有基础**；
- checkpoint-on-idle、跨 runtime restore、统一状态 manifest 和 policy/provenance 是**需要继续补齐的实现**。

### 1.4 三者在 CPU 和内存弹性上的当前边界

| 系统 | 活跃时 CPU | 活跃时内存 | idle 资源释放 | vCPU/CPU 运行态扩容 |
|---|---|---|---|---|
| AKernel 当前 | gVisor 进程受 cgroup v1 request/limit 约束 | 进程/内核直接参与 host reclaim；没有 VM balloon | keep-alive 保状态但占内存；delete 可释放但丢失运行态 | 容器理论上可改 cgroup quota，但当前公开 SDK/sandboxd 没有资源 patch API |
| Substrate gVisor | WorkerPool Pod 固定 shape | runsc sandbox 占 Worker 内存 | 显式 checkpoint/delete sandbox，归还 Worker | 没有 per-Actor CPU/RAM patch |
| Substrate microVM | CH 启动时固定 vCPU | guest RAM 固定；当前无 balloon 控制闭环 | 显式 snapshot 后 teardown VMM，归还 Worker | 创建时 BootVcpus 等于 MaxVcpus；没有接入 CPU hotplug |
| AgentENV | Firecracker 启动前固定 vCPU；Paused 后无 vCPU thread | 固定 guest MemTotal；free-page reporting 可部分释放 host backing | pause + diff snapshot + stop FC 可彻底释放私有 CPU/RSS | 不支持把 1-vCPU sandbox 原地 hotplug 到 4 vCPU |

AgentENV 是三者当前唯一明确接入“运行中的 VM 部分内存归还”机制的系统：virtio-balloon free-page reporting 能让 guest 已释放页对应的 host backing 被丢弃，降低 Firecracker RSS。它不改变 guest MemTotal，不等于缩小 sandbox 的 logical reservation，也不能替代真正 idle 时的 pause/snapshot/stop。

## 2. 比较方法与证据边界

### 2.1 为什么要区分设计、代码和实验

本文采用三档证据：

| 证据等级 | 可得出的结论 |
|---|---|
| 运行时/API 源码和部署清单 | 当前版本确实存在的机制、参数和边界 |
| README、architecture、roadmap | 项目定位、演示声明和未来方向；若与源码不一致，以源码为准 |
| AKernel 论文草稿 | 研究问题、抽象和计划评测；不能自动证明已经实现或已经达到性能目标 |

例如 AKernel README 中 40 ms cold start、fork-based launch 和 checkpoint/restore 后面有星号，并明确说明未包含在 v0.1.0；Substrate README 的 sub-second 和 30x+ oversubscription 来自 demo 描述；AgentENV README 的高密度方向也需要在指定服务器上复测。本文不会把这些数字混入结果表。

### 2.2 “公平”不等于强行抹平所有系统差异

三者的优化对象并不相同：

- AKernel 当前优化集群 sandbox 服务和 lazy rootfs；
- Substrate 优化长期逻辑 Actor 到预热 Worker 的复用、路由和唤醒；
- AgentENV 优化 Firecracker sandbox、memory/rootfs snapshot layering 和恢复 I/O。

实验既要提供共同 workload，也要保留每套系统的 native optimized mode。正确做法是同时发布：

1. common-denominator 配置，用相同镜像、资源、工作负载和存储；
2. native-best 配置，允许开启 distill-fs/Dragonfly、golden snapshot/request parking、OverlayBD/P2P/warm pool；
3. 对每项优化给出组件分解，而不是把所有差异压成一个总分。

## 3. AKernel：论文设计、当前源码和实现缺口

### 3.1 论文提出的 cloud-OS 抽象

AKernel 论文认为最小管理对象不应是短命 Pod 或无状态函数，而应是一个跨节点保持逻辑身份的有状态执行环境。四个核心抽象是：

- **AProc**：数据中心进程。具有稳定身份，携带执行、资源、runtime、状态和策略上下文，可以 materialize、run、wait、checkpoint、restore、fork 和 terminate。
- **ARealm**：跨节点的隔离、资源、权限和核算域。它不是单机 cgroup，而是覆盖一个 session、任务、RL rollout 或 batch 的预算、并发、SLO、credential、egress 和 audit boundary。
- **APlane**：镜像、代码、只读对象、可变工作目录、checkpoint 和 policy metadata 的统一命名、版本和 lineage 视图。
- **ASyscall**：面向 Agent 的稳定生命周期原语，包括 Spawn、Invoke、Fork、Checkpoint、Restore、Realm、Wait、Hibernate 和 Terminate。

论文希望把一次调用组织成：

~~~text
Agent / Agent Runtime
  -> ASyscall Gateway
  -> ARealm admission / quota / policy
  -> global scheduler：资源 + 状态位置 + runtime compatibility + policy
  -> node-local Sandbox Daemon：本地缓存 + 瞬时负载 + runtime fast path
  -> APlane：image / object / file delta / process-memory checkpoint
~~~

这比“启动一个 sandbox”更宽。其主要性能指标也不是 VMM process ready，而是从请求开始到镜像、依赖、数据、凭证、网络策略和工具服务全部可用的 time-to-useful-action。

### 3.2 论文中的 idle 与资源弹性设计

论文设计将 Running 和 Waiting 分开：

- 短等待保留 runtime/cache；
- 长等待或节点压力高时 checkpoint-on-idle；
- checkpoint 后释放 CPU 和 memory；
- 请求到来时根据 checkpoint locality、runtime compatibility 和 ARealm policy 选择节点恢复；
- 决策考虑 checkpoint 写成本、预计等待时间、restore SLO 和节点压力。

这是一种正确的分层资源观：

~~~text
活跃：完整 CPU + RAM
短等待：降 CPU entitlement，保内存和热缓存
长等待：checkpoint，释放 compute
再次到达：按状态局部性恢复
~~~

但当前论文没有给出固定阈值、预测模型、代价函数或实测结果，所以它仍是待实现和待验证的策略。

### 3.3 当前源码真正实现了什么

当前公开源码的数据路径可以概括为：

~~~text
akernel_sdk.Sandbox
  -> openYuanRong backend adapter
  -> cluster scheduling / instance lifecycle
  -> node sandboxd
     -> runsc / gVisor
     -> cgroup v1
     -> veth + iptables
     -> rootfs manager
        -> local / OCI / S3 raw / Nydus
        -> distill-fs FUSE lazy read
        -> local chunk cache + LMDB dedup
        -> optional peer chunk service / Redis ownership index
~~~

已经存在的用户能力包括：

- create/close/delete sandbox；
- command、background process、filesystem、PTY；
- port forwarding 和 reverse tunnel；
- CPU millicores、memory MiB 的 request/limit；
- stable name、detached lifecycle 和指定 node affinity；
- cluster resources 查询；
- local、OCI、Nydus、S3-backed rootfs 和额外挂载；
- standalone、Helm、Alibaba Cloud ACK/Huawei Cloud CCE Terraform；
- OTEL、Prometheus、Grafana、Loki 和 Tempo 部署资产。

distill-fs 是当前最接近论文 APlane 数据路径的组件：

- raw image 和 Nydus RAFS v5 按需读取；
- sparse local cache + bitmap；
- 4 MiB chunk content-addressed dedup；
- 本地/peer chunk serving；
- 静态 peer 或 Redis discovery；
- Redis-backed chunk ownership index；
- touched chunk 后台持久化和 cache hole punching。

它解决的是 immutable/rootfs working set 的 lazy materialization 和跨环境复用；它还不是把可变文件、进程内存、策略引用和 checkpoint 原子绑定起来的完整 APlane。

### 3.4 论文—源码差距矩阵

| 论文能力 | 当前源码状态 | 判断 | 不补齐的直接影响 |
|---|---|---|---|
| AProc 稳定身份 | named/detached sandbox、node affinity、openYuanRong instance handle | 部分实现 | 还不能跨 checkpoint/restore/migration 维持完整 lineage |
| ARealm 跨节点预算/策略域 | 有 cluster scheduler、JWT 和资源字段，但没有独立 ARealm API/metadata | 尚未形成公开抽象 | 无法按一次 Agent task 统一 quota、cost、egress、credential 和 audit |
| APlane 统一版本化状态 | distill-fs lazy rootfs、cache/dedup/peer；SDK 初始化还设置 bypass_datasystem | 部分数据路径 | 镜像、工作目录、memory checkpoint、policy 不能形成同一个可恢复 commit |
| ASyscall | create/invoke/delete 等 SDK 操作 | 部分实现 | Fork、Checkpoint、Restore、Hibernate 等论文语义不可调用 |
| 两级调度 | openYuanRong + node sandboxd 架构存在 | 部分实现 | 当前代码证据不足以证明资源、状态局部性、runtime 和 policy 的联合代价模型 |
| per-sandbox CPU/memory | SDK 和 sandboxd proto 支持 request/limit | 已实现基础配额 | 没有运行态 resource update API，也没有阶段感知自动调额 |
| 多 runtime 自动选择 | runtime abstraction 存在，但 SDK 只接受 runsc | 未实现闭环 | 不能按隔离、checkpoint、兼容性或成本在 gVisor/microVM 间选择 |
| fork-based launch | README roadmap | 未实现 | 无法测 running-state fork、branch latency 和 COW sharing |
| checkpoint/restore | README roadmap；sandboxd API 只有 Start/Delete/Wait/List/Stats/ListAvailableRuntimes | 未实现 | idle 环境只能保留运行态或删除重建，不能无损 scale-to-zero |
| checkpoint-on-idle | SDK 有 idle_timeout，但它是 lifecycle reclaim，不是状态 checkpoint | 未实现 | 长等待时面临“保状态占资源”和“释放资源丢状态”二选一 |
| lazy rootfs | distill-fs raw/Nydus 按需 read | 已有实质实现 | 仍需 benchmark touched bytes、first-read 和高 fanout external traffic |
| P2P | distill-fs peer chunk；部署可选 Dragonfly且默认关闭 | 部分实现 | 需要明确内建 peer 与 Dragonfly 的职责、去重和调度 locality |
| policy/provenance | 论文设计有 policy reference 和 audit timeline | 缺少完整公开实现证据 | 无法验证 fork/restore 后重新授权和外部 side-effect 可追踪 |
| 大规模 evaluation | 论文只列计划；源码有 create pressure 和 file copy benchmark | 尚无论文结果 | 目前不能声称分钟到秒、万级并发或成本收益 |

### 3.5 AKernel 当前 CPU/内存语义

AKernel SDK 的 CPU 单位是 millicores，memory 单位是 MiB；还能给出 cpu_limit 和 mem_limit。sandboxd 把资源交给 cgroup v1 和 gVisor runtime。

这和 microVM 的固定虚拟硬件不同：

- CPU quota/weight 在 Linux cgroup 层理论上可以动态修改，不需要 CPU hotplug；
- 但当前公开 Sandbox SDK 和 sandboxd protobuf 没有 UpdateResources/Resize API，因此产品层仍是创建时固定；
- gVisor 不是每个 sandbox 固定预分配一整块 guest RAM 的 Firecracker/CH 模型，host 可以按普通进程内存压力 reclaim 或 swap；
- 只要进程状态仍存活，匿名 working set、runsc 开销和 rootfs/cache 状态仍会占 host 资源；
- idle_timeout 回收实例后可以释放资源，但不能无损保留进程树、内存、PTY 和连接状态。

所以论文的 checkpoint-on-idle 对 AKernel 不是“小优化”，而是完成其资源弹性闭环的关键功能。

## 4. Agent Substrate：Actor/Worker 多路复用

### 4.1 核心抽象

Substrate 把逻辑 Actor 和物理 Worker 分离：

- Actor 是可 suspend/resume、可移动的逻辑实例；
- Worker 是 Kubernetes 中预热的 Pod slot；
- 一个 Worker 当前同一时刻承载一个 Actor；
- Actor 数量可以远大于 Worker 数量，只有 Running Actor 占 slot；
- WorkerPool、ActorTemplate 和 SandboxConfig 是低频 Kubernetes CRD；
- Actor、Worker assignment 和 snapshot 等高频状态放在 ValKey/Redis；
- ate-api-server 处理生命周期和 workflow；
- atelet DaemonSet 协调节点和 snapshot 搬运；
- ateom-gvisor 或 ateom-microvm 驱动具体 runtime；
- atenet DNS/Envoy/atunnel 提供 location-transparent routing。

CreateActor 的结果首先是逻辑 SUSPENDED Actor。ResumeActor claim 一个空闲 Worker、恢复 snapshot 并等待 ready；SuspendActor checkpoint 到 durable object storage 后归还 Worker；PauseActor 可以保留 node-local snapshot 和 locality。

### 4.2 为什么它与 AKernel 论文相似

两者共同认识到：

- 逻辑 Agent identity 不应绑定 Pod/节点；
- 高频生命周期不应每次经过 Kubernetes scheduler；
- idle state 应与物理 compute slot 分离；
- 请求到 suspended execution domain 时应自动或显式唤醒；
- snapshot locality 应参与放置。

Substrate 的 Actor 很接近 AProc；Atespace/WorkerPool 触及 ARealm 的一部分；snapshot metadata/object store 触及 APlane 的一部分。但 Substrate 当前没有 AKernel 论文中统一的 task-level resource/policy/provenance 语义，也没有让 Actor 选择任意 per-instance CPU/RAM 的通用 resource context。

### 4.3 两种 sandbox class

| class | runtime | snapshot | restore |
|---|---|---|---|
| gvisor | runsc | process tree + filesystem checkpoint | runsc restore |
| microvm | Kata guest + Cloud Hypervisor | CH VM state + sparse memory-ranges；guest tmpfs upper 也随 RAM 保存 | CH OnDemand + userfaultfd demand paging |

microVM snapshot 的 sparse delta 会与 restore source 合并，生成 self-contained complete memory-ranges 再上传。它会利用 sparse extent 避免空洞 I/O，但当前不是 AgentENV 那种跨 checkpoint 保留 parent OverlayBD layer chain 的组织方式。

### 4.4 idle、CPU 和内存边界

- 当前通用 auto-suspend-on-idle 尚未实现，核心路径由用户或 framework 显式调用 SuspendActor/PauseActor。
- 请求到 suspended Actor 时，router 可以 ResumeActor；WorkerPool 暂时满时有有界 request parking、retry 和 singleflight。
- per-Actor CPU/memory 不是一等 API；资源 shape 主要由 WorkerPool Pod requests/limits 和 SandboxConfig 决定。
- microVM 创建时 BootVcpus 等于 MaxVcpus，源码没有调用 CH CPU hotplug。
- 当前 microVM 配置没有 AgentENV 等价的 virtio-balloon/free-page reporting 控制闭环。
- 真正释放 Worker 资源依赖 snapshot 后销毁 gVisor sandbox 或 CH VMM，而不是运行态逐步缩小一台 VM。

### 4.5 当前成熟度

Substrate README 明确将项目标为 early development、not ready for production，并提示 API 可能变化。它已经有完整 quickstart、demo、Locust benchmark harness 和 microVM 资产脚本，但 benchmark README 自称 nascent suite，仓库没有一张可直接当论文结果引用的稳定结果表。

## 5. AgentENV：Firecracker 与增量状态数据面

### 5.1 当前架构

AgentENV 的主要路径是：

~~~text
E2B-compatible HTTP API / aenv CLI
  -> per-node orchestrator
  -> Firecracker sandbox
     -> fixed pre-boot vCPU / memory
     -> netns + veth + iptables
     -> envd / MMDS
     -> rootfs and extra drives: OverlayBD -> ublk
     -> memory resume: OverlayBD layer stack -> read-only ublk -> FC mmap/COW
  -> committed snapshot repository: POSIX or object storage
  -> optional multi-node gateway/scheduler and P2P
~~~

与 Substrate 相比，AgentENV 没有固定的 Actor→Worker Pod slot 抽象；每个 Running sandbox 对应一个独立 Firecracker 进程。它的 warm pool 更底层，可以预热 network slot、Firecracker process 和某些 ublk device。

### 5.2 pause/snapshot/resume

AgentENV pause 的关键步骤：

1. orchestrator 将 sandbox 从 Running 转为 Pausing；
2. 暂停 Firecracker；
3. Firecracker native Diff snapshot 生成 vm_state.bin 和 sparse mem.bin；
4. present pages 转成新的 OverlayBD memory layer，与 parent memory layers stack；
5. rootfs 和 attached-drive writable upper 被 close/seal/restack 成新 lower；
6. paused metadata 持久化；
7. 停止 Firecracker 并释放网络 runtime、vCPU thread 和私有 guest RSS。

resume 时：

1. 取得或启动 Firecracker process；
2. 重建 rootfs/extra-drive ublk；
3. 从 memory layer stack 创建或共享 read-only memory ublk；
4. 以 Firecracker file-backed memory backend load snapshot；
5. 页面首次读取走 ublk→OverlayBD；写入通过 COW 形成本实例匿名页。

多个 sandbox 从同一 snapshot/template 启动时，可以引用同一个 memory ublk，并利用 host Linux page cache 共享只读 memory image。

### 5.3 CPU 与内存弹性

AgentENV 的 CPU 规格在 Firecracker pre-boot machine config 中设置。当前没有 vCPU hotplug API，也不能把 1-vCPU snapshot 原地恢复为 4 vCPU；要换规格通常需要创建另一台 VM，且 snapshot topology compatibility 仍需处理。

内存有两级机制：

1. **运行态部分回收**：virtio-balloon free-page reporting 把 guest 已释放页通知给 VMM，Firecracker 对对应 host backing 执行丢弃，从而降低 RSS。guest 仍看到原 MemTotal，logical quota/reservation 也不变。
2. **idle 全量释放**：pause + incremental snapshot + stop Firecracker，释放私有匿名内存、vCPU thread 和 runtime。memory image 和 rootfs state 保存在可恢复层中。

因此 AgentENV 同时具备“轻度 pressure reclaim”和“完整 hibernate”两档，但没有 active balloon target、virtio-mem hotplug 或 vCPU hotplug 的资源控制闭环。

### 5.4 多节点边界

AgentENV 已有 Go gateway/scheduler、节点 heartbeat、sandbox binding、Kubernetes discovery 和 P2P artifact index，但该分布式控制面仍明确是 prototype：

- 调度策略以 round-robin/random 为主；
- resource-aware admission/locality-aware scheduling 还不完整；
- binding 默认是内存态，Redis HA 只覆盖部分 existing-sandbox data plane；
- request parking、per-tenant fairness 和大规模 simultaneous wakeup 仍是缺口。

单节点 Firecracker/OverlayBD/ublk 路径目前比多节点控制面成熟，第一阶段 benchmark 应先测单节点。

## 6. 三者多维对比

### 6.1 总体能力矩阵

| 维度 | AKernel 当前源码 | Agent Substrate | AgentENV |
|---|---|---|---|
| 主要抽象 | Sandbox/instance；论文目标是 AProc/ARealm/APlane/ASyscall | Actor、Worker、WorkerPool、ActorTemplate、Atespace | Sandbox、Template、Committed Snapshot、Node |
| 用户 API | Python SDK；commands/files/PTY/proxy/tunnel | gRPC + kubectl-ate + CRD + Actor DNS | E2B-compatible HTTP/OpenAPI + aenv CLI + proxy |
| runtime | gVisor/runsc | gVisor 或 Kata+Cloud Hypervisor | Firecracker |
| Kubernetes | standalone 可不依赖；集群模式可用 Helm | 硬依赖 K8s | 单节点不依赖；多节点有 K8s manifests |
| 活跃实例映射 | 一个 sandbox 对应一个 runsc workload | 大量 Actor→少量预热 Worker；一个 Worker 同时一个 Actor | 一个 Running sandbox→一个 FC process |
| per-instance CPU/RAM | 有 request/limit | 主要是 WorkerPool fixed shape，没有通用 per-Actor shape | 有 cpuCount/memoryMB |
| 运行态 vertical resize | 公开 API 无 | 公开 API 无；CH 无 hotplug 闭环 | 无 vCPU/memory hotplug |
| lazy rootfs | distill-fs raw/Nydus FUSE | golden snapshot；gVisor/CH 各自准备 rootfs | OverlayBD/ublk；overlaybd-native image 可 remote read |
| rootfs cache/dedup | sparse cache、4 MiB chunk dedup、peer/Redis、可选 Dragonfly | OCI cache/object store；P2P/更深 locality 在 roadmap | OverlayBD layers、page cache、可选 Iroh P2P |
| full execution checkpoint | 当前没有公开 API | gVisor/CH 均有 | FC diff snapshot |
| incremental memory persistence | N/A | sparse delta merge成 self-contained complete image | parent OverlayBD layer chain |
| demand-paged memory restore | N/A | CH OnDemand/userfaultfd | FC file-backed mmap + ublk |
| running VM 部分内存释放 | N/A，属于 host process reclaim | 当前无 balloon 闭环 | free-page reporting 已接入 |
| idle scale-to-zero | delete/reclaim 会丢 execution state | 显式 suspend/pause 后释放 Worker | pause/snapshot/stop FC |
| auto idle detector | idle_timeout 是生命周期回收，不是 checkpoint-on-idle | 通用 auto-idle 尚未实现 | 显式 pause、lease/expiry eviction；proxy 可触发 resume |
| fork/clone | roadmap | 可从 snapshot 创建 Actor，但不是 live running fork | 有 sandbox fork/snapshot/template |
| 请求唤醒 | 无完整 checkpoint resume 路由 | Actor DNS + Envoy + atunnel + request parking | proxy 根据 sandbox ID 路由并可 resume |
| 分布式调度 | openYuanRong 集群后端 | 原生 K8s+Redis 专用 Actor control plane | gateway/scheduler prototype |
| 策略/provenance | 论文目标宽，当前尚未形成完整 ARealm/APlane 实现 | 有 identity/mTLS/NetworkPolicy 基础，仍 early | microVM/netns/API auth/custom extension，policy plane 较弱 |
| 当前最适合验证 | gVisor create/exec、lazy rootfs、cluster deployment | multiplexing、resume/suspend、parking | microVM snapshot、lazy memory restore、layer sharing、balloon |

### 6.2 共同点

1. 都把 Agent 环境视为比普通无状态函数更长寿、有 mutable state 的执行对象。
2. 都试图把高频生命周期操作从通用 Kubernetes Pod creation 路径中移出或缩短。
3. 都重视镜像/rootfs 工作集而不是无条件全量拉取。
4. 都希望 logical identity 与瞬时物理位置解耦。
5. 都需要在 burst 和长 idle 之间权衡 latency、资源占用和状态持久性。
6. 当前都没有完整的运行态 CPU vertical scaling 产品闭环。

### 6.3 本质差异

#### 差异一：AKernel 论文的系统边界最宽

AKernel 希望把 task-level realm、object/state plane、runtime selection、policy 和 provenance 一起纳入 kernel abstraction。Substrate 和 AgentENV 当前更专注 sandbox/actor substrate。这个宽边界可能形成 AKernel 的论文差异化，但也意味着必须用源码和实验把抽象真正闭合。

#### 差异二：Substrate 主要复用 Worker slot

Substrate 的密度来自逻辑 Actor 数远多于物理 Worker 数，idle Actor 是 snapshot。它把 routing、parking 和 Worker claim 做成集群一等对象。AKernel 当前创建 gVisor instance；AgentENV 每个 Running sandbox 有自己的 Firecracker process，主要复用的是更底层 warm components 和 snapshot data。

#### 差异三：AgentENV 把 memory 和 rootfs 做成分层 block data plane

AgentENV 的 parent layer、shared memory ublk 和 page-cache reuse 特别适合频繁 checkpoint、同模板 fanout 和 snapshot divergence。Substrate 当前更像可移动的 self-contained snapshot object；AKernel 当前的 distill-fs 深在 immutable rootfs working set，还没有 process-memory snapshot layer。

#### 差异四：CPU/RAM 资源 shape 的归属不同

- AKernel：Sandbox create request；
- Substrate：WorkerPool Pod/runtime class；
- AgentENV：每个 Firecracker sandbox。

这会直接影响 benchmark。若给 Substrate 一个 4-vCPU WorkerPool，不能把它误写成“每个 Actor 动态请求 4 vCPU”；若给 AgentENV 1 vCPU，也不能期待运行中 hotplug 到 4。

#### 差异五：状态正确性边界不同

VM/process snapshot 只能保存本地执行状态，不能自动让数据库写、外部 API、代码提交和用户可见 output 具备 exactly-once。AKernel 论文的 provenance/policy 方向有机会覆盖这一层，但当前三套系统都需要上层 operation ID、fencing、commit protocol 和 side-effect ledger。

## 7. 服务器部署可行性

### 7.1 单节点部署矩阵

| 系统/模式 | 能否部署 | 关键前置 | 工程风险 | 建议用途 |
|---|---|---|---|---|
| AKernel standalone | 能 | Linux x86-64、cgroup v1、Docker/Pouch、privileged container、iptables | 中；cgroup v1 是主要限制 | 三者共同 create/exec 和 AKernel lazy rootfs |
| AKernel Helm | 能 | 兼容 K8s、cgroup v1 worker、storage/registry 配置 | 中高 | 多节点和 observability |
| Substrate gVisor on kind | 能 | Go、kubectl、Docker、kind；ValKey、RustFS 由脚本部署 | 中 | Actor/Worker、gVisor、parking |
| Substrate microVM on kind | 有条件能 | 上述条件 + /dev/kvm + matching-arch CH/Kata/virtiofsd/kernel/rootfs assets + privileged worker | 高 | 与 AgentENV 的 microVM 对照 |
| AgentENV Docker/systemd | 能 | Linux 6.8+、/dev/kvm、ublk_drv、root setup、privileged/CAP_NET_ADMIN/CAP_SYS_ADMIN、iptables/netns | 中 | Firecracker 单节点、snapshot、balloon |
| AgentENV Kubernetes | 能，但控制面是 prototype | KVM/ublk-enabled worker、privileged DaemonSet、gateway/scheduler | 中高 | 多节点 smoke 和后续 scale |

### 7.2 推荐服务器拓扑

最低可行拓扑：

~~~text
DUT：同一台裸金属服务器
  64+ physical cores, 256+ GiB RAM, local NVMe, x86-64, hardware virtualization
  每次只运行 AKernel / Substrate / AgentENV 之一

LOADGEN：独立服务器
  发送 API、HTTP、PTY 和并发负载
  采集 client-side monotonic timestamps

DATA：独立服务器或隔离盘/网卡
  同一个 OCI registry
  同一个 S3-compatible object store
  可选 Redis/ValKey
~~~

如果只有一台机器，可以把 loadgen 和 object store 放在 DUT，但必须给它们固定 CPU、memory 和 I/O budget，并在结果中扣除；否则控制面和数据服务自身资源会污染 sandbox 数据。

### 7.3 同机顺序部署的兼容性方案

推荐预检：

~~~bash
uname -a
test -r /dev/kvm
test -e /dev/ublk-control
stat -fc %T /sys/fs/cgroup
docker info
lscpu
numactl --hardware
lsblk
~~~

不要先假设 Ubuntu 24.04 默认配置能运行 AKernel v0.1.0。可选方案：

- 为 AKernel 准备 cgroup v1 boot profile；
- 为 AgentENV 准备官方推荐的 Ubuntu 24.04/Linux 6.8+ profile；
- 每次切换系统后重新执行 host preflight；
- 不复用上一个系统留下的 iptables、network namespace、ublk device、Docker network、page cache 和 object-store bucket；
- 用相同 BIOS、CPU governor、SMT、NUMA、kernel mitigation 和 NVMe firmware 设置。

如果不能接受重启/重装，使用两台同型号服务器并让每套系统在两台上都跑，最终采用 host fixed-effect 或归一化结果，而不是“AKernel 在 A 机、AgentENV 永远在 B 机”。

### 7.4 各系统 smoke 路径

以下命令只说明仓库已有路径，本次调研没有执行。

AKernel standalone：

~~~bash
cd /mnt/u/hukeyang/AKernel/akernel/deploy/standalone
./start.sh
~~~

Substrate gVisor：

~~~bash
cd /mnt/u/hukeyang/AKernel/agent-substrate/substrate
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system
hack/install-ate-kind.sh --deploy-demo-counter
~~~

Substrate microVM：

~~~bash
cd /mnt/u/hukeyang/AKernel/agent-substrate/substrate
hack/run-microvm-demo.sh
~~~

AgentENV Docker：

~~~bash
cd /mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV
sudo bash scripts/docker-setup.sh
docker run --rm -it --device /dev/kvm --privileged -v /dev:/dev \
  -p 8000:8000 ghcr.io/kvcache-ai/aenv-server:latest
~~~

### 7.5 部署成功判据

不能只以 process/Pod Running 为成功。每套系统应完成：

1. control plane health；
2. 创建一个最小环境；
3. 执行或服务第一条有意义请求；
4. 写入并读取 mutable state；
5. 暴露端口并从独立 loadgen 访问；
6. 获取 CPU/memory/latency metrics；
7. Substrate/AgentENV 额外验证 snapshot 后状态连续；
8. 清理后 host 无残余 sandbox PID、mount、netns、veth、iptables 和 ublk device。

## 8. 公平 benchmark 设计

### 8.1 统一 workload image

建议构建一个最小 agentbenchd OCI image，并在三个系统使用同一 digest。它提供：

| endpoint/动作 | 用途 |
|---|---|
| /readyz | 统一 readiness 和 time-to-useful-action 终点 |
| /counter | 检查 process memory 在 suspend/resume 后是否连续 |
| /cpu?n= | 固定指令/计算量，测 active CPU entitlement |
| /memory?bytes=&mode=touch/free | 触碰、释放匿名页，测 RSS、snapshot 和 balloon |
| /fs/write、/fs/read、/fs/fsync | 固定 mutable disk delta |
| /tree | 创建子进程、FD、Unix socket 和 PTY，检查 checkpoint coverage |
| /state/hash | 对进程、文件和配置生成 correctness digest |

相同 image 不代表相同启动机制：Substrate 会从 golden snapshot 恢复，AgentENV 可从 template/snapshot 恢复，AKernel 当前只能 fresh create 或 warm image create。因此每个结果必须标注 materialization mode。

### 8.2 实验 Track A：三者共同的 fresh-to-useful-action

统一定义：

~~~text
T0：client 发出逻辑环境创建请求
T1：control plane 接收
T2：物理 runtime 开始创建/恢复
T3：workload readyz 成功
T4：第一条有意义请求完成

TTUA = T4 - T0
~~~

各系统映射：

- AKernel：Sandbox create → command/service ready → first response；
- Substrate：CreateActor → ResumeActor → routed first response；
- AgentENV：CreateSandbox → envd/workload ready → proxy/command first response。

Substrate CreateActor 本身只创建 suspended logical record，不能只测这个 RPC 就宣称“启动完成”。结果至少拆成 logical create、worker claim、snapshot/rootfs materialize、runtime restore、ready、routing 六段。

矩阵：

- concurrency：1、8、32、128；
- image：small、1 GiB dependency image、5–10 GiB sparse/toolchain image；
- cache：cold registry/object/local cache、warm node cache；
- resources：1 vCPU/2 GiB、2 vCPU/4 GiB、4 vCPU/8 GiB；
- samples：每个稳态 cell 至少 100，cold cell 至少 30 个独立 cache reset round。

### 8.3 Track B：AKernel gVisor vs Substrate gVisor

这是最接近“同 runtime”的对照。应固定：

- 同一 runsc release；
- 同一 OCI image digest；
- 同一 host kernel；
- 同一 CPU/memory cgroup shape；
- 同一网络模式和安全设置；
- 同一 registry/object store；
- 同样关闭或开启 lazy/cache 优化。

重点回答：

1. AKernel openYuanRong→sandboxd 与 Substrate Actor→Worker claim 的 control-plane 差异；
2. fresh gVisor create 与 golden snapshot restore 的 latency 差异；
3. distill-fs lazy rootfs 对 touched bytes 和 cold TTUA 的收益；
4. Substrate WorkerPool/request parking 对 burst tail 和 admission failure 的收益；
5. idle Actor 超额订阅下逻辑环境数、active slot 数和 host memory 的关系。

需要注意 Substrate 的 gVisor backend 可能依赖带特定 checkpoint networking flag 的 runsc；若无法与 AKernel 固定到同一 binary，应把 runtime version 当独立变量，不再称为严格 same-runtime。

### 8.4 Track C：Substrate CH vs AgentENV Firecracker

这是一项 microVM **完整系统**对比，不是相同 VMM benchmark。必须显式报告：

- Cloud Hypervisor vs Firecracker；
- guest kernel/config；
- vCPU topology；
- guest memory；
- CH userfaultfd OnDemand vs FC file-backed mmap+ublk；
- Substrate self-contained sparse memory-ranges vs AgentENV OverlayBD parent layers；
- rootfs writable upper 的位置；
- snapshot storage、compression 和 network；
- warm Worker/FC pool 状态。

工作负载矩阵：

- dirty anonymous memory：0、128 MiB、1 GiB、4 GiB；
- mutable disk delta：0、100 MiB、1 GiB；
- checkpoint count：1、5、20、100；
- restore fanout：1、8、32、128 sandbox from one snapshot；
- cache：cold object store、warm local, same-template shared page cache；
- measure：pause time、snapshot commit time、bytes written、resume-to-ready、first-fault tail、post-resume steady throughput、host RSS/PSS。

这个 Track 能直接验证两种数据面差异：

- Substrate 每轮 self-contained snapshot 是否在频繁 checkpoint 下产生更高 upload/merge 成本；
- AgentENV parent layer chain 是否在链很长时增加 lookup/compaction 成本；
- shared memory ublk/page cache 是否改善同模板 fanout；
- userfaultfd 与 ublk/mmap 在随机 working set 下的 first-fault latency。

### 8.5 Track D：idle resource elasticity

统一 workload：

1. 环境启动；
2. 触碰 1 GiB anon memory，写 512 MiB mutable disk；
3. 运行 counter 并打开 PTY/FD；
4. idle 30 秒、5 分钟、30 分钟；
5. 再发请求并验证 state hash。

比较模式：

| 系统 | 模式 |
|---|---|
| AKernel | keep running；delete/recreate 作为 state-losing baseline |
| Substrate | explicit SuspendActor/ResumeActor；PauseActor local variant |
| AgentENV | keep running + balloon free-page reporting；Pause/Resume full hibernate |

AKernel delete/recreate 不能伪装成 checkpoint restore。它可以作为成本下界或状态丢失 baseline，但 correctness 列必须标为失败/N/A。

测量：

- idle host CPU-seconds；
- host RSS/PSS、cgroup memory.current 或 v1 等价值；
- page cache、swap、KVM memory；
- snapshot logical/physical bytes；
- object-store/network bytes；
- state hash correctness；
- resume TTUA p50/p95/p99/p99.9；
- 在固定 1 秒或 5 秒恢复 SLO 下可承载的逻辑环境数。

### 8.6 Track E：CPU 和内存 shape

CPU：

- 为 1、2、4 vCPU/CPU-equivalent 分别创建新实例；
- 跑固定 CPU-time 和并行度 1/2/4 的 workload；
- 测 guest/app throughput、host CPU-seconds、steal/throttling 和 tail；
- 不测试不存在的 1→4 hotplug，除非以后实现 resource update。

内存：

- touch 25%、50%、100% guest/limit；
- 应用 free 后继续运行；
- 对 AgentENV 观察 free-page reporting 前后 FC RSS；
- 对 Substrate microVM 观察当前无 balloon 时 guest reservation/RSS；
- 对 AKernel 观察进程 RSS/cgroup reclaim；
- 再进入 full suspend/pause，测最终可回收量。

这样可以把“部分 reclaim”和“完整 hibernation”分开，避免将 RSS 下降误写成 guest memory size 已缩小。

### 8.7 Track F：镜像与状态数据局部性

至少包含四种场景：

1. 所有 cache cold；
2. 同节点 warm；
3. 目标节点 miss、peer 命中；
4. object/registry 远端且受限带宽。

指标：

- registry external bytes；
- object-store bytes；
- peer/P2P bytes；
- touched image bytes；
- first-open/first-page latency；
- page/chunk cache hit ratio；
- node disk physical bytes；
- 同镜像 128-way fanout 的峰值带宽和 TTUA。

AKernel 应分开测 raw、Nydus、distill-fs dedup、Dragonfly；AgentENV 应分开测标准 OCI conversion 和 overlaybd-native remote ref；Substrate 应分开 gVisor 与 microVM/golden snapshot。native format 实验不能与 common OCI baseline 混成一个数字。

### 8.8 Track G：多节点与故障

第二阶段再做三节点以上：

- burst placement；
- cache/snapshot locality；
- cross-node restore；
- Worker/node drain；
- control-plane restart；
- object store latency/failure；
- concurrent resume 同一 Actor/sandbox；
- node loss 后 state correctness。

当前 AKernel 没有 execution checkpoint，跨节点 stateful restore 记为 N/A；AgentENV multi-node control plane 是 prototype；Substrate 的 portable SuspendActor 更适合先作为 reference。

### 8.9 统计与结果呈现

每项至少报告：

- p50、p90、p95、p99、p99.9；
- mean 仅作补充；
- success/failure/timeout/correctness rate；
- 95% bootstrap confidence interval；
- client latency和server trace breakdown；
- host CPU-seconds、RSS/PSS、I/O、network 和 storage bytes；
- 所有失败样本，不静默丢弃；
- commit、kernel、VMM、runsc、image digest 和 config hash。

建议每个 Track 单独一张结果表，未支持功能填 N/A，不给三者生成一个未经解释的综合分数。

## 9. 建议的执行顺序

### Phase 0：环境与 benchmark contract

1. 固定硬件、BIOS、kernel 和 cache reset 方法；
2. 完成 agentbenchd image；
3. 为三套 API 写薄 adapter；
4. 定义 TTUA、pause、snapshot commit、resume 和 correctness 时间点；
5. 对 host metrics 和 distributed trace 校时；
6. 冻结全部 commit/config/image digest。

### Phase 1：单节点 smoke

1. AKernel standalone create/exec/file/PTY；
2. Substrate kind gVisor counter 和 sandbox demo；
3. AgentENV Docker create/exec/snapshot/resume；
4. Substrate microVM counter continuity；
5. 每次完整 teardown 和 residue audit。

### Phase 2：共同能力和同 runtime

1. Track A 三者 fresh TTUA；
2. Track B AKernel gVisor vs Substrate gVisor；
3. Track F common OCI cold/warm；
4. active CPU/memory isolation和burst create。

### Phase 3：microVM 和 state elasticity

1. Track C Substrate CH vs AgentENV FC；
2. Track D idle；
3. Track E memory partial reclaim；
4. snapshot chain、fanout、P2P/cache。

### Phase 4：AKernel 补实现后

1. AKernel gVisor checkpoint/restore；
2. checkpoint-on-idle；
3. APlane state manifest；
4. cross-node restore；
5. 三者共同 stateful idle elasticity。

## 10. AKernel 不具备 checkpoint-on-idle 的影响

### 10.1 资源效率出现二选一

没有 full execution checkpoint 时：

- keep alive：进程、runsc、匿名 working set、FD/PTY 和部分 cache 保留，恢复快但密度受限；
- delete/recreate：释放资源，但进程内 memory、terminal、连接和未持久化状态丢失，后续还要重建环境。

Agent 工作负载大量时间在等待模型、外部工具或人类输入，这个二选一会放大成本。

### 10.2 无法实现论文 AProc 生命周期

论文定义 Checkpointed 和 Restoring 是 AProc 的核心状态。若源码只有 Start/Delete：

- AProc identity 无法与 execution state 一起跨节点；
- migration 不能表示为 checkpoint→placement→restore；
- fork 不能共享运行时基线；
- stable name 只能定位逻辑 instance，不能证明恢复同一执行状态。

### 10.3 APlane 无法形成一致 checkpoint

只保存 lazy rootfs/cache 不够。一个可恢复版本至少需要绑定：

- immutable image digest；
- mutable rootfs/workspace delta；
- process tree 与 anonymous memory；
- runtime/kernel/CPU compatibility；
- attached objects/volumes；
- policy references；
- lineage、epoch、digest 和 commit status。

如果这些对象各自成功但没有 durable commit barrier，恢复可能看到“旧内存 + 新文件”或“新文件 + 无 policy authorization”。

### 10.4 调度器缺少 state-aware decision 的真实输入

论文希望比较本地等待与远程加载成本。没有 checkpoint metadata 和实际 populated bytes，调度器无法准确知道：

- state 在哪里；
- 恢复需要多少网络和 I/O；
- 哪个 runtime 能恢复；
- CPU/kernel feature 是否兼容；
- restore 是否满足 deadline。

因此 locality-aware scheduling 会停留在镜像 cache 层，覆盖不了完整 execution state。

### 10.5 benchmark 和论文论证不闭合

当前 AKernel 可以与两者比较 fresh create、lazy image、active performance；但不能参加 state-preserving idle、resume、fork 和 snapshot efficiency。论文若将 checkpoint-on-idle 作为主要贡献，就必须补实现和结果，否则对照表会在最关键列显示 N/A。

## 11. AKernel 建议补齐路线

### P0：先建立 runtime capability contract

为每个 runtime 显式声明：

- full/process/data snapshot；
- incremental；
- lazy restore；
- cross-node portability；
- CPU/memory resize；
- balloon/reclaim；
- fork；
- multi-volume；
- kernel/CPU/device compatibility；
- privilege/security requirements。

调度器按 capability 选择，不只按 runtime 字符串。

### P1：扩展 sandboxd 生命周期 API

建议增加：

- Pause；
- Checkpoint；
- Restore；
- ForkFromCheckpoint；
- UpdateResources；
- InspectCapabilities。

返回值要包含 request ID、operation epoch、snapshot ID、状态、错误类型和是否可安全重试。

### P2：先实现 gVisor checkpoint/restore

这是投入产出比最高的第一步：

- 当前已有 runsc adapter；
- 可直接与 Substrate gVisor 做 same-runtime 对照；
- 能先闭合 AProc idle lifecycle；
- 可把 mutable rootfs 与 process checkpoint 纳入 snapshot manifest。

要处理 networking、PTY/FD coverage、connected socket、runsc version pin、checkpoint compatibility、cleanup 和 security sanitization。

### P3：建立 APlane snapshot manifest 和原子提交

推荐顺序：

1. freeze；
2. 写 file delta/runtime checkpoint；
3. 校验 digest；
4. durable data commit；
5. 发布 immutable manifest；
6. 更新 AProc pointer/epoch；
7. 释放 runtime；
8. 恢复只读取最后一个 committed epoch。

policy/credential 只存可重新验证的 handle，不复制 secret。

### P4：实现 idle policy 和请求唤醒

至少包括：

- explicit Wait/Hibernate；
- idle detector；
- keep-warm、throttle、checkpoint 三档；
- checkpoint cost 与预计等待阈值；
- resume singleflight；
- bounded request parking；
- admission、deadline、tenant fairness；
- snapshot locality-aware placement。

### P5：再引入 microVM runtime

不建议一开始同时追 Firecracker 和 Cloud Hypervisor。先定义 adapter contract，再选一个：

- 若重视与 AgentENV 数据面复用，研究 Firecracker + layered memory/rootfs；
- 若重视与 Substrate runtime 对照，研究 Kata+CH/userfaultfd；
- 若只需更强隔离，先实现固定 shape 和 full hibernation，不把 vCPU hotplug 作为首要目标。

CPU elasticity 的短期实用方案是固定最大 topology、host cgroup 动态调 entitlement；真正 hotplug 要额外处理 guest online、snapshot ABI、interrupt state 和 accounting。

### P6：补齐论文评测

论文最少需要：

- real Agent traces，而非只有 synthetic sandbox；
- active/wait/idle 分布；
- time-to-useful-action breakdown；
- burst fanout；
- touched bytes/P2P external traffic；
- checkpoint-on-idle cost model；
- agent goodput、cost/successful task、resource-hours saved；
- policy/provenance overhead；
- failure/correctness。

## 12. 预期假设：需要实验验证，不能当作结果

以下是根据架构可以提出的假设，而不是本次调研的实测结论：

1. AKernel distill-fs 可能在大镜像、小 working set、cold cache 下减少 TTUA 和 external bytes。
2. Substrate 可能在逻辑 Actor 数远大于 active 数、WorkerPool 已预热时提供更好的高密度和 burst backpressure。
3. AgentENV 可能在同 snapshot 大 fanout、频繁 incremental checkpoint 和 running guest free-page 场景中获得更好的 memory/storage sharing。
4. Substrate self-contained snapshot 可能在少量 checkpoint 时管理简单，但在高 checkpoint 频率下写放大更明显。
5. AgentENV layer chain 可能在初期减少写入，但长链需要观测 lookup、cache 和 compaction 成本。
6. AKernel 当前没有 checkpoint，因此 keepalive 可能有最低 resume latency但最高 idle memory，delete/recreate 则相反且无法保持完整状态。

正式文档或论文必须用 Track A–G 的数据支持或否定这些假设。

## 13. 最终建议

### 13.1 是否值得部署比较

值得。三者恰好覆盖三个互补方向：

- AKernel：更宽的 Agent-native cloud-OS 研究命题和 lazy rootfs/集群 substrate；
- Substrate：Actor/Worker multiplexing、Kubernetes 集成和请求唤醒；
- AgentENV：Firecracker + incremental state data plane 和运行态部分内存回收。

### 13.2 最有信息量的第一轮实验

优先顺序：

1. 在同一裸金属服务器顺序部署三者，完成 Track A；
2. 固定 runsc 做 AKernel vs Substrate gVisor；
3. 做 Substrate CH vs AgentENV FC snapshot/fanout；
4. 用 idle workload 展示 AKernel 当前 keep/delete 的能力缺口；
5. 实现 AKernel gVisor checkpoint 后重新加入同一 idle test。

### 13.3 论文叙述应如何表述

在补实现以前，建议写成：

> AKernel 提出 AProc/ARealm/APlane/ASyscall 和 checkpoint-on-idle 的统一设计；当前开源原型已经实现基于 openYuanRong、gVisor sandboxd 和 distill-fs 的可部署 sandbox substrate，而 full execution checkpoint、跨 runtime restore、统一 APlane commit 和阶段感知 idle policy 正在补齐。

不要写成这些功能已经全部实现，也不要引用尚未产生的万级并发、分钟到秒或资源节省数字。

## 14. 关键源码索引

### 14.1 AKernel 论文

| 主题 | 本地证据 |
|---|---|
| time-to-useful-action、状态与 idle 问题 | [1-introduction.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/1-introduction.tex:70) |
| 四抽象概述 | [1-introduction.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/1-introduction.tex:90) |
| AProc 生命周期 | [4-abstractions.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/4-abstractions.tex:9) |
| ARealm | [4-abstractions.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/4-abstractions.tex:44) |
| APlane | [4-abstractions.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/4-abstractions.tex:58) |
| ASyscall | [4-abstractions.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/4-abstractions.tex:77) |
| 两级调度 | [5-design.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex:9) |
| Lazy Load + Dragonfly | [5-design.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex:32) |
| APlane checkpoint/restore/fork | [5-design.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex:62) |
| checkpoint-on-idle | [5-design.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex:82) |
| runtime adapter | [5-design.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex:95) |
| 计划性 implementation | [6-implementation.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/6-implementation.tex:1) |
| 计划性 evaluation | [7-evaluation.tex](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/7-evaluation.tex:1) |

### 14.2 AKernel 当前源码

| 主题 | 本地证据 |
|---|---|
| 当前能力、roadmap 与星号说明 | [README.md](/mnt/u/hukeyang/AKernel/akernel/README.md:33) |
| SDK 当前 openYuanRong backend | [SDK README](/mnt/u/hukeyang/AKernel/akernel/sdk/python/README.md:1) |
| 只支持 runsc、资源/idle/name/node 参数 | [sandbox.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/sandbox.py:91) |
| openYuanRong options 与 bypass_datasystem | [_openyuanrong.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:42) |
| sandboxd API 只有 Start/Delete/Wait/List/Stats | [sandbox-api.proto](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:22) |
| template_id 当前开源路径边界 | [sandbox-api.proto](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:87) |
| gVisor/cgroup/network/rootfs 架构 | [sandboxd README](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/README.md:1) |
| gVisor/cgroup v1/TLS 限制 | [sandboxd README](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/README.md:72) |
| distill-fs 当前能力 | [distill-fs README](/mnt/u/hukeyang/AKernel/akernel/src/distill-fs/README.md:1) |
| distill-fs read path | [distill-fs README](/mnt/u/hukeyang/AKernel/akernel/src/distill-fs/README.md:333) |
| standalone 部署 | [standalone README](/mnt/u/hukeyang/AKernel/akernel/deploy/standalone/README.md:1) |
| cgroup v1 部署限制 | [deployment README](/mnt/u/hukeyang/AKernel/akernel/deploy/README.md:28) |

### 14.3 Agent Substrate

| 主题 | 本地证据 |
|---|---|
| 定位、early development、quickstart | [README.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/README.md:5) |
| Actor/Worker 和 K8s 关系 | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:25) |
| CRD 与 Redis 动态状态拆分 | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:205) |
| gVisor/microVM sandbox class | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:313) |
| lifecycle 和显式 SuspendActor | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:352) |
| WorkerPool resources | [workerpool_types.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/pkg/api/v1alpha1/workerpool_types.go:22) |
| ActorTemplate/SandboxConfig | [api-guide.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/api-guide.md:90) |
| microVM KVM 和资产要求 | [api-guide.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/api-guide.md:226) |
| kind KVM passthrough | [create-kind-cluster.sh](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/hack/create-kind-cluster.sh:47) |
| microVM asset runbook | [microVM assets README](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/hack/microvm-assets/README.md:1) |
| CH vCPU 固定 Boot=Max | [run.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateom-microvm/run.go:546) |
| CH checkpoint/merge | [checkpoint.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateom-microvm/checkpoint.go:40) |
| CH OnDemand restore | [restore.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateom-microvm/restore.go:110) |
| request parking | [request-parking.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/request-parking.md:1) |
| benchmark suite | [benchmarking README](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/benchmarking/README.md:1) |

### 14.4 AgentENV

| 主题 | 本地证据 |
|---|---|
| 产品定位和部署前置 | [README.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/README.md:18) |
| 单节点 quickstart | [quickstart.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/getting-started/quickstart.md:1) |
| Docker 部署 | [docker.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/deployment/docker.md:1) |
| Kubernetes 部署 | [kubernetes.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/deployment/kubernetes.md:1) |
| 总体架构与 memory restore | [architecture.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/internals/architecture.md:1) |
| memory ublk/page-cache sharing | [architecture.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/internals/architecture.md:128) |
| pause/resume orchestration | [service.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/orchestrator/service.rs:973) |
| Firecracker snapshot | [sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:511) |
| file-backed memory restore | [instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:505) |
| memory/rootfs OverlayBD snapshot | [overlaybd_snapshot.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/overlaybd_snapshot.rs:457) |
| shared memory ublk | [device.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/ublk/device.rs:449) |
| balloon free-page reporting | [instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:387) |
| proxy auto-resume | [proxy.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/api/proxy.rs:649) |
| 多节点 prototype | [architecture.md](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/docs/src/internals/architecture.md:215) |

## 15. 相关既有调研

- [AgentENV Firecracker Idle CPU/内存释放与恢复机制源码分析](./20260728T073510Z-agentenv-firecracker-idle-cpu-memory-survey.md)
- [AgentENV 缺少 vCPU 弹性的影响与 Firecracker 运行态内存部分回收机制分析](./20260728T090135Z-agentenv-vcpu-elasticity-firecracker-live-memory-reclaim-survey.md)
- [Google AX / Agent Substrate 架构与 AgentENV 源码级对比调研](./20260801T081103Z-google-ax-agent-substrate-vs-agentenv-survey.md)
