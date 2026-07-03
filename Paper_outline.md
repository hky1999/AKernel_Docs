# AKernel 论文大纲

## 暂定题目

**AKernel: An Agent-Native Cloud Operating System for Datacenter Use**

备选中文题目：**AKernel: 面向 Datacenter Use 的 Agent-Native 云操作系统**

## 一句话主张

AI Agent 正在从“使用一台虚拟电脑”走向“驱动整个数据中心”。
AKernel 要证明：Agent 时代需要一种新的云操作系统，把执行、状态、对象局部性、安全策略、审计 provenance 和集群构建统一成 Agent 可调用、系统可优化、运维可观测的执行内核。

## 核心论点

人工智能应用正在从被动问答走向高阶自主化。以 OpenClaw、Manus 以及内部 Hermes/OpenClaw 类 workload 为代表的智能应用，已经开始自主理解目标、分解任务、寻找资源、选择工具，并在云端执行代码、访问数据、部署服务、诊断系统和生成产物。在软件工程、Agentic RL、自动化运维和数据分析等场景中，云计算基础设施的直接使用者正在从人类和固定服务，转变为大量动态创建、长期交互、会产生副作用的 Agent。

AI sandbox API 正在商品化。E2B、Modal、Daytona、OpenSandbox、Cube Sandbox、Kubernetes/GKE Agent Sandbox 等系统已经说明：创建一个可执行命令、读写文件、配置网络、保存快照的 sandbox，正在变成基础能力。因此，AKernel 不能把“启动一个 sandbox 并在里面运行 agent”作为论文创新点。真正的系统问题是：如何让云端成千上万个有状态、突发式、产物密集、易空闲、受策略约束的 Agent 执行单元，像操作系统调度进程一样，被调度、恢复、迁移、挂起、审计和回收。

我们要通过真实 workload trace 证明，Agent 用云有 5 个与传统 FaaS、batch job、microservice 都不同的特征：

1. **Stateful**：Agent 的工作区、进程树、浏览器 profile、package cache、checkpoint、外部对象引用和中间产物共同构成执行状态。
2. **Bursty and branchy**：训练、评测、并行探索和回滚会在短时间内启动大量相似但不完全相同的执行环境。
3. **Artifact-heavy**：代码仓库、日志、测试结果、数据集、模型产物、网页快照和 provenance log 是主要输出，也是后续调度输入。
4. **Idle-prone**：Agent 经常等待 LLM、外部 API、人审、网络、构建和测试；空闲期间继续占用 sandbox 会浪费大量资源。
5. **Policy-constrained**：Agent 会运行不可信代码、访问凭证、调用外部服务并产生不可回滚副作用，网络、数据、凭证和审计策略必须跟着执行走。

AKernel 的核心抽象是：**Agent Process** 运行在 **Agent Realm** 中，读写 **Agent Object**，并通过 **Datacenter Function** 被 Agent 像调用函数一样驱动。论文要把这 4 个抽象写成 OS 语义，而不是 API 名词。

## OS 类比

这张表可以放在 Introduction 或 Design Overview 中，用来帮助读者快速理解 AKernel 的定位。注意论文正文里不要过度依赖类比，后续必须用 trace 和机制把每个类比坐实。

| AKernel | 传统 OS | 论文中要证明的新语义 |
| --- | --- | --- |
| 物理集群 / 多云 region | 物理机 | Agent 不关心单机，而是按任务构建可用的执行域 |
| 集群内节点 | NUMA node | 调度要感知 image、checkpoint、object、policy locality |
| Agent Realm | 用户 / 进程组 / namespace | 一个任务或会话的状态域、资源域、权限域和审计域 |
| Agent Process | 进程 | 可调度、可挂起、可恢复、可迁移、带策略的云端执行单元 |
| Sandbox runtime | 进程隔离机制 | gVisor、Kata、Firecracker、nanovisor 等是下层实现选择 |
| 容器镜像 | 程序 / binary | Agent execution 的 base environment |
| 容器镜像缓存 | Page cache | 热层、依赖、工具链和浏览器 profile 影响 useful start |
| 镜像 lazy load | Demand paging | 只 hydrate first useful action 所需 working set |
| Agent Object | 文件 / named object | 文件、checkpoint、日志、数据集、package cache、trace、产物统一进入调度 |
| Datacenter Function | syscall / fork-exec / RPC | 表面像函数调用，实际展开为调度、对象放置、生命周期和策略执行 |

## Background: 生产场景与需求

这一节的目标不是泛泛介绍 Agent，而是解释 AKernel 从什么真实场景中长出来。建议后续用内部数据替换 `TODO`。

### B1. ACompany 的 Agent Sandbox 部署形态

**训练侧 workload：Agentic RL / 蒸馏 / 大规模评测。**
训练阶段会根据 rollout、evaluation 或 distillation 阶段，瞬时启动上千到上万个 sandbox instance。不同任务可能使用不同镜像、依赖、代码仓库、测试集、浏览器状态或工具链。需要补充的数据包括：

- `TODO`: 单次训练/evaluation burst 的 sandbox 数量分布。
- `TODO`: 镜像大小、实际 touched bytes、package cache 命中率。
- `TODO`: sandbox duration、active/idle ratio、checkpoint/retry/branch 次数。
- `TODO`: 每个 realm 产生的 artifact 数量和大小。

**Serving 侧 workload：OpenClaw / Hermes / 内部 Agent 应用。**
Serving 阶段由用户任务、工作流事件或外部系统触发，要求在短时间内提供可交互的执行环境。任务可能包含代码修改、命令执行、数据分析、浏览器操作、云资源诊断、构建测试和产物提交。需要补充的数据包括：

- `TODO`: request arrival burstiness、p50/p95/p99 session duration。
- `TODO`: first useful action latency breakdown。
- `TODO`: LLM wait、tool execution、external API wait、人审 wait 的占比。
- `TODO`: 失败、重试、人工接管、恢复和迁移比例。

### B2. ACompany 对 Agent Sandbox Infra 的需求

1. **快速部署到算力所在 region。**
   Sandbox 集群和存储系统需要跟着 CPU/GPU/NPU 资源走，在香港、马来西亚、新加坡等 region 快速搭建和扩缩容。论文里可把这条抽象成 Build Your Own Cluster：Agent runtime 不应被固定在单一云、单一 region 或单一 Kubernetes 集群中。

2. **弹性扩缩容与成本控制。**
   系统需要根据整体 CPU/memory/GPU 供给、任务 burst、空闲等待和 checkpoint 状态实时扩缩容。关键不是最大 throughput，而是 SLO 内单位成本完成的正确任务数，即 agent goodput。

3. **多 runtime 选择。**
   不同 workload 对兼容性、启动速度、隔离强度和资源开销要求不同。AKernel 需要在 gVisor、Kata、Firecracker、nanovisor 等 runtime 之间选择，并把 runtime selection 纳入 Agent Process 生命周期，而不是暴露为用户手工配置。

4. **高并发 useful start。**
   生产系统关心的不是 sandbox boot 完成，而是 repo、依赖、对象、网络策略、凭证和 tool server 已经 ready，并且 Agent 能完成第一条有意义动作。AKernel 应将 `time-to-useful-action` 作为核心指标。

5. **强隔离、受控访问和可审计。**
   Agent 会运行不可信代码并操作真实数据、凭证和外部系统。网络 egress、secret injection、object ACL、人审 gate、不可回滚 side effect 和 provenance log 必须成为 OS 语义。

## Problem Statement

AKernel 论文应避免写成“我们提供一个更好的 sandbox SDK”。更准确的问题陈述是：

> Existing systems give each agent a sandbox, a tool runtime, or a serverless substrate. AKernel treats large numbers of stateful, artifact-producing, policy-carrying agents as cloud processes, and provides an agent-native cloud OS that co-designs lifecycle, placement, object locality, checkpoint/restore, policy, provenance, and cluster deployment.

本文默认 LLM serving 是外部服务，不把模型推理、KV cache 和 batching 作为主贡献。AKernel 优化的是 Agent 周围的执行底座：sandbox 放置、状态恢复、对象局部性、资源弹性、安全策略和审计。

## Motivation

### M1. Computer Use 太小了

Computer Use 给 Agent 一台虚拟电脑。Datacenter Use 给 Agent 一个可以按任务构建、按状态调度、按策略约束、按成本回收的云端执行域。

论文应把这个转变写成三层：

- 单机 computer use：Agent 控制浏览器、桌面、终端或代码解释器。
- Cloud sandbox use：Agent 在远端隔离环境中运行命令、读写文件、保存快照。
- Datacenter use：Agent 需要弹性算力、分布式对象、跨节点状态、并发分支、策略控制和多 region 部署。

### M2. Sandbox 必要，但已经不够

已有 AI sandbox 平台已经覆盖 sandbox lifecycle、exec、file API、network policy、snapshot、pause/resume、BYOC 或 Kubernetes integration。AKernel 的 novelty 不能停留在这些 primitive 上。

这一节应该放一张能力对比表：

| System | Sandbox lifecycle | Exec / FS | Network / secret policy | Snapshot | K8s / BYOC | Agent realm/object/locality/provenance |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| E2B-like sandbox | yes | yes | partial | yes | partial | no public OS semantics |
| Modal-like sandbox | yes | yes | yes | yes | partial | limited |
| Daytona-like computer | yes | yes | yes | yes | yes | limited |
| OpenSandbox / Cube | yes | yes | yes | yes | yes | runtime-centric |
| Kubernetes Agent Sandbox | yes | yes | K8s-native | yes | yes | sandbox-centric |
| AKernel | yes | yes | yes | yes | yes | target contribution |

结论句：

> Once sandboxes become abundant, the missing problem is not how to create one sandbox, but how to manage many agent execution domains as OS objects.

### M3. Agent Workload 既不是 FaaS，也不是 Microservice

Agent workload 与 FaaS 的区别：

- 不是单次 stateless invocation，而是多轮、多工具、多状态的 session。
- 不是函数返回后状态消失，而是 workspace、artifact、checkpoint 和外部引用持续存在。
- 不是只优化 handler cold start，而是优化 first useful action 和 end-to-end task goodput。

Agent workload 与 microservice 的区别：

- 不是长期稳定 replica，而是短时 burst、分支探索、反复恢复和迁移。
- 不是静态 API 服务，而是会动态创建进程、访问工具、修改代码、生成文件和调用外部系统。
- 资源需求不是稳定 request/limit，而是在 tool-call、build/test、browser、LLM wait、human wait 等 phase 之间剧烈变化。

### M4. Useful Start，而不只是 Cold Start

一次 Agent execution 的关键路径可以拆成：

```text
T_useful_start =
  T_realm_admission
+ T_scheduler_placement
+ T_runtime_start_or_restore
+ T_image_working_set
+ T_dependency_or_tool_warmup
+ T_object_or_checkpoint_hydration
+ T_policy_and_secret_setup
+ T_first_useful_action
```

一个 shell ready 的 sandbox 未必 useful。对于 coding agent，useful action 可以是目标 repo 中 `git status` 成功返回，或首个测试/构建命令开始产生有效输出。对于 browser agent，useful action 可以是 browser profile 和页面状态 ready 后首个 DOM action 成功。对于 ops agent，useful action 可以是 kubeconfig、网络策略和只读凭证就绪后首个诊断命令成功。

### M5. 状态生命周期是 Agent 的核心 OS 问题

Agent 状态不是只有 filesystem snapshot。至少包括：

1. base image / runtime environment
2. dependency and package cache
3. workspace filesystem diff
4. running process tree and memory state
5. browser / PTY / tool server state
6. external object references
7. provenance and audit log
8. credential and network policy handles

AKernel 要把 `checkpoint`、`restore`、`fork`、`hibernate`、`migrate`、`commit`、`abort` 变成 Agent Process 生命周期原语，并说明哪些状态可重放、哪些必须持久化、哪些必须 scrub、哪些需要人审或补偿。

### M6. Data Locality 和 Policy 是调度输入

传统调度器主要看 CPU、memory、quota、affinity 和 priority。Agent 调度还需要看：

- image layer 和 package cache 是否在本节点。
- repo、checkpoint、browser profile、dataset、log 等 Agent Object 在哪里。
- object 的 reuse distance、fanout/fanin、confidentiality 和 mutability。
- 该节点或 region 是否满足 egress、secret、tenant isolation 和审计要求。
- 当前 action 是否 deterministic、idempotent、credential-bearing、external-side-effect 或 requires-human-approval。

因此，AKernel 的 scheduler 应该调度 Agent Realm，而不只是 Pod、container 或 function。

## Workload Characterization Plan

论文成败取决于这一节。最小 trace 需要覆盖 realm/session/process/object/tool/policy/resource 多层。

| Trace 层级 | 必须采集的字段 | 支撑的论文论点 |
| --- | --- | --- |
| Agent Realm | session duration, status, success metric, retry count, human intervention, cost, SLO | Agent task 是整体生命周期，不是单次 invocation |
| Agent Process | create/start/fork/checkpoint/restore/hibernate/migrate/commit/fail, runtime, node, image, resources | Agent execution 需要 OS lifecycle |
| Agent Object | object type, size, producer/consumer, reuse distance, cache hit, node/region location, read/write count | 状态和产物是一等对象，调度要跟着状态走 |
| Tool/Event | tool type, shell/file/network/browser/LLM/API duration, input/output size, side effects | tool-call boundary 是资源、策略、checkpoint 的关键边界 |
| Resource | CPU, memory, disk I/O, network, GPU, active/idle/wait timeline | Agent 资源需求 phase-aware，而不是 container-level stable |
| Policy/Provenance | credential use, egress domain, allow/block/review, artifact refs, command refs | Agent 执行需要 capability 和 audit 贯穿生命周期 |

### 预期 Workload Insights

1. **Agent sessions are longer-lived and more stateful than functions.**
2. **Burst scaling is common in training, evaluation, and parallel exploration.**
3. **Artifacts and checkpoints dominate state movement and storage pressure.**
4. **A large fraction of wall-clock time is idle or externally blocked.**
5. **Useful start is dominated by multiple layers, not only sandbox boot.**
6. **Object locality and policy compatibility materially affect goodput and cost.**

### 建议图表

- Figure 1：Agent session timeline，标出 LLM thinking、sandbox active、external wait、checkpoint、restore、artifact commit。
- Figure 2：CDF of session duration，与 FaaS invocation 或 batch task 对比。
- Figure 3：active/idle/wait ratio，展示 checkpoint-on-idle 的资源节省机会。
- Figure 4：artifact/checkpoint size distribution，展示 Agent Object 是主要状态载体。
- Figure 5：image size vs touched bytes，证明完整拉镜像不是合理默认。
- Figure 6：time-to-useful-action breakdown。
- Figure 7：object locality hit/miss 对端到端 latency 和 cross-node traffic 的影响。

## 核心抽象

### Agent Process

AKernel 中最小的可调度 Agent 执行单元：一个有状态、强隔离、携带策略、可观测、可挂起恢复的云端进程。

Agent Process 包含：

- compute context：CPU、memory、GPU、browser、runtime、image。
- execution context：commands、tool calls、process tree、open services、PTY。
- state context：filesystem diff、memory snapshot、checkpoint、object references。
- policy context：network egress、credential capability、data access scope、human gate。
- provenance context：prompt、tool call、命令、文件变更、网络访问和产物之间的因果关系。

生命周期原语：

```text
create -> useful_start -> run -> checkpoint/hibernate/fork/migrate -> restore -> commit/abort -> destroy
```

### Agent Realm

Agent Realm 是一个 Agent 任务、多 Agent workflow、RL rollout、evaluation batch 或用户会话的隔离域、状态域、权限域和资源域。

Realm 管理：

- 一组 Agent Processes。
- 一组共享或私有 Agent Objects。
- checkpoint lineage 和 branch graph。
- resource budget、SLO、cost accounting。
- credential/network/data policy。
- provenance log 和 audit boundary。

### Agent Object

Agent Object 是 Agent 产生、消费、共享和持久化的统一对象。它可以是代码仓库快照、filesystem diff、checkpoint、terminal log、browser trace、dataset、model artifact、package cache、test output、policy decision 或 provenance span。

Object metadata 应进入调度：

| Metadata | Examples | Scheduling meaning |
| --- | --- | --- |
| type | repo, checkpoint, package cache, browser profile, log | 决定 hydration 和 cache 策略 |
| location | node, rack, region, object store | 决定 compute-to-data 或 data-to-compute |
| mutability | immutable, COW, append-only, external side effect | 决定 fork/commit/rollback |
| confidentiality | public, tenant-private, credential-bearing | 决定可共置范围 |
| reuse distance | same realm, same user, task family, global | 决定 warm retention |
| hydration SLA | eager, lazy, background, first-action-critical | 决定 useful start 关键路径 |

### Datacenter Function

Datacenter Function 是面向 Agent 的调用语义：表面上像函数调用，实际展开为一个或多个 Agent Processes、对象依赖、调度决策、sandbox 生命周期、状态管理和策略执行。

示意：

```python
run = akernel.submit(
    image="agent-env:latest",
    task="evaluate 10k candidate patches",
    resources={"cpu": 8, "mem": "32Gi"},
    objects=[repo, tests, package_cache],
    policy=capabilities,
    lifecycle="checkpoint-on-idle",
)
result = run.await_result()
```

API 不是贡献本身。贡献是背后的 OS 语义：状态可挂起、artifact 可复用、权限可审计、资源可回收、调度可利用 locality。

## 设计原则

1. **Useful start，而不只是 cold start**：衡量到达第一次有意义动作的时间。
2. **状态是一等公民**：checkpoint、restore、fork、migrate、hibernate、commit 是生命周期原语。
3. **计算跟着状态走**：调度要感知 image cache、checkpoint location、object locality 和 package/browser cache。
4. **策略跟着执行走**：网络、凭证、数据访问、人审和审计策略绑定到 Agent Process 与 Agent Realm。
5. **Goodput 优先于 throughput**：优化 SLO 内单位成本完成的正确任务数。
6. **快路径与审计路径分离**：高频 start/fork/restore/cgroup update 走节点快路径，provenance 和控制面状态异步持久化但不能丢。
7. **Build Your Own Cluster**：Agent 应能在算力所在 region 构建和操作自己的执行集群。

## 系统架构

### Control Plane

SDK、CLI、Gateway、Realm manager、admission controller、scheduler、metadata service。

核心设计点：

- 以 Agent Realm 为 admission 和 accounting 单元。
- 两级调度：Realm scheduler 做 placement/fairness/locality/policy，node daemon 做高频生命周期操作。
- 与 Kubernetes / Helm / Terraform 集成，但不把每个 tool-call 都放到 K8s API Server critical path 上。

### Execution Plane

Sandbox Daemon、AFaaS、nanovisor、runtime selection、process lifecycle、phase-aware resource control。

核心设计点：

- 根据 workload 在 gVisor、Kata、Firecracker、nanovisor 等 runtime 之间选择。
- 支持 create、fork、restore、hibernate、migrate、commit、destroy。
- 将 cgroup/eBPF/resource update 对齐到 tool phase、active/wait phase 和 checkpointable boundary。

### Data Plane

openYuanRong Data System、Agent Object、KV/Object storage、node-local cache、cross-node transfer、shared memory。

核心设计点：

- Agent Object 不只是存储 API，而是调度和恢复输入。
- object locality、reuse distance、confidentiality、mutability 进入 scheduler。
- 支持 realm-local、node-local、region-local 和 remote object placement。

### Image Plane

distill-fs lazy loading、Dragonfly P2P distribution、image cache locality、package cache、browser/tool cache。

核心设计点：

- 优化 first useful action 所需 working set，而不是完整 image pull。
- Burst scaling 时避免 registry/object-store thundering herd。
- 将 image layer locality 与 object/checkpoint locality 联合考虑。

### State Plane

Checkpoint/Restore、hibernation、fork/explore/commit、restore locality、failure recovery。

核心设计点：

- checkpoint-on-idle 释放资源。
- fork 支持并行探索和 RL rollout。
- commit/abort 处理 workspace diff 和 external side effect。
- restore 与 object hydration、policy scrub、provenance replay 绑定。

### Policy and Network Plane

eBPF NAT、双向代理、capability enforcement、credential injection、network egress policy、human approval gate、audit/provenance。

核心设计点：

- policy 是执行上下文，不是部署后的附加规则。
- external side effect、credential-bearing action、non-idempotent action 需要显式标注。
- provenance log 记录 prompt、command、file diff、network access、object lineage 和 policy decision。

### Deployment Plane

基于 Terraform + Helm 的多云 Build Your Own Cluster。

核心设计点：

- 按 region 快速部署 sandbox 集群、object/cache 层和控制面接入点。
- 支持跟随算力供给移动，降低跨 region 数据移动和等待时间。
- 论文中要评估从空 region 到可服务 Agent workload 的部署时间、人工步骤和稳定性。

### Observability Plane

Agent span、tool span、OS resource metrics、object events、policy decisions、checkpoint events、cost accounting。

核心设计点：

- 将 LLM/tool trace 与 OS/cgroup/object/network trace 对齐。
- 用同一套 trace 支撑 debugging、evaluation、resource control 和 workload characterization。

## 关键机制

### K1. Realm-aware Scheduling

调度单位从 Pod/Function/Job 提升到 Agent Realm。Scheduler 同时考虑：

- resource availability
- image/package/browser cache locality
- object/checkpoint locality
- policy compatibility
- tenant isolation
- SLO/cost budget
- branch/fanout/fanin pattern

### K2. Useful-start Optimized Startup Pipeline

启动路径按 first useful action 排序：

1. admission 和 policy check
2. locality-aware placement
3. runtime create or restore
4. critical image blocks lazy load
5. repo/checkpoint/package/browser objects hydrate
6. network/secret/proxy setup
7. tool server / PTY / browser / LSP readiness
8. first useful action completion

### K3. Checkpoint-on-idle and Branch Lifecycle

当 Agent 等待 LLM、外部 API、人审或长时间 I/O 时，AKernel 将 process hibernate 或 checkpoint，并释放 CPU/memory。对于并行探索，AKernel 从同一 checkpoint fork 多个 branch，最后 commit 成功分支或 abort 失败分支。

### K4. Object-aware Restore and Migration

恢复不是简单把 sandbox 拉到任意节点。AKernel 会根据 checkpoint、workspace diff、package cache、repo、browser profile 和目标 region 的 locality 选择恢复位置，必要时预取或迁移关键 Agent Objects。

### K5. Policy-carrying Execution

Agent Process 的权限随 lifecycle 移动。网络、secret、object ACL、human gate、audit retention 与 Agent Realm 绑定，restore/migrate/fork 后必须重新验证或 scrub。

## 主要贡献

### C1. Agent Workload Characterization

基于生产 trace 证明 Agent workload 是 stateful、bursty、artifact-heavy、idle-prone、policy-constrained，并量化 useful start、object locality、checkpoint、idle resource waste 和 policy/provenance 的系统影响。

### C2. Agent-Native Cloud OS Abstractions

提出 Agent Process、Agent Realm、Agent Object 和 Datacenter Function，并定义对应的 lifecycle、state、policy、provenance 和 scheduling semantics。

### C3. AKernel 跨层系统实现

基于 openYuanRong、AFaaS、Data System、distill-fs、Dragonfly、Checkpoint/Restore、eBPF NAT、Terraform 和 Helm，实现 agent-aware scheduling、useful-start pipeline、object-aware restore、checkpoint-on-idle 和 policy-carrying execution。

### C4. 端到端评估与生产经验

在真实 Agent workload 上评估 task goodput、cost-to-solution、time-to-useful-action、resource-hours saved、object locality、burst scaling、policy overhead 和 Build Your Own Cluster 部署效率。

## 论文结构

### 1. Introduction

- 从 Computer Use 到 Datacenter Use。
- AI sandbox API commodity 化，AKernel 不能只做 sandbox SDK。
- 真实 Agent workload 暴露 state、artifact、idle、policy、locality 问题。
- AKernel 的 cloud OS 主张和贡献。

### 2. Background and Related Systems

- Cloud OS / distributed execution：SigmaOS、YuanRong、Ray、Borg/Omega。
- Serverless and stateful serverless：Cloudburst、Beldi、Pocket、SONIC、ExoFlow。
- Cold start / sandbox lifecycle：Firecracker、SEUSS、FaaSnap、Catalyzer、AFaaS、Dirigent。
- AI sandbox platforms：E2B、Modal、Daytona、OpenSandbox、Cube Sandbox、Kubernetes/GKE Agent Sandbox。
- Agent OS / resource control / checkpoint：AgentCgroup、Crab、DeltaBox、Fork-Explore-Commit、AgentTrust 等。

本节结论：已有系统分别解决了 sandbox、serverless、workflow、state、resource control 的一部分，但缺少以 Agent Realm / Process / Object 为中心的 cloud OS 语义。

### 3. Agent Workload Characterization

- Trace collection methodology。
- Production workloads：training/evaluation、serving、coding、ops、data/browser tasks。
- 5 个 workload insight。
- 对系统设计的要求：realm scheduling、useful start、state lifecycle、object locality、policy provenance。

### 4. AKernel Abstractions

- Agent Process。
- Agent Realm。
- Agent Object。
- Datacenter Function。
- Lifecycle semantics。
- State and policy semantics。

### 5. System Design

- Control plane。
- Execution plane。
- Data/object plane。
- Image plane。
- State plane。
- Policy/network/provenance plane。
- Deployment and observability plane。

### 6. Implementation

- 如何整合 openYuanRong、AFaaS、Data System、distill-fs、Dragonfly、eBPF NAT、Terraform 和 Helm。
- Runtime selection 和 sandbox daemon。
- Object metadata 和 locality-aware scheduler。
- Checkpoint/restore/fork/hibernate 的实现边界。
- Policy/provenance instrumentation。

### 7. Evaluation

- Workload trace 结果。
- Time-to-useful-action breakdown。
- Burst scaling and useful start。
- Checkpoint/hibernate resource savings。
- Object locality and cross-node traffic。
- Runtime selection and isolation overhead。
- Policy/provenance overhead。
- Build Your Own Cluster。
- Ablation study。

### 8. Production Experience

- 内部部署经验。
- Agentic RL、FaaS、Spark、Sandbox workload 的共存。
- 多 region 部署和小团队运维。
- AI-assisted debugging。
- 失败案例：object locality miss、checkpoint inconsistency、policy misconfiguration、runtime fallback。

### 9. Discussion

- GPU/NPU 支持和 LLM serving 的边界。
- Windows/macOS/Android sandbox。
- Replay、rollback 和不可逆外部 side effect 的限制。
- 安全边界和 threat model。
- 与闭源 sandbox 平台的对比限制。
- AKernel 是否应成为 Kubernetes extension、cloud OS、还是 Agent runtime substrate。

## Evaluation Plan

### Workloads

- Coding agent tasks：repo checkout、patch、test、artifact commit。
- Agentic RL rollout and evaluation：high fanout、branch、checkpoint、reward logging。
- Data analysis tasks：dataset/object access、notebook/script execution、result artifacts。
- Browser/web tasks：browser profile、DOM/screenshot artifacts、network policy。
- Terminal/ops tasks：kubeconfig/secret policy、diagnostic commands、audit。
- Mixed workloads：FaaS、Spark、sandbox tasks 共存，测试 isolation 和 scheduling。

### Baselines

- Kubernetes + containerd/gVisor。
- Kubernetes Agent Sandbox-like design。
- openYuanRong baseline。
- E2B-like single-sandbox baseline。
- Cube/OpenSandbox-like multi-node sandbox baseline。
- AKernel without checkpoint/hibernate。
- AKernel without locality-aware scheduling。
- AKernel without lazy image/P2P distribution。
- AKernel without policy-aware placement。

### Metrics

- time-to-useful-action。
- p50/p95/p99 task latency。
- successful tasks per hour。
- cost per successful task。
- agent goodput：successful tasks / dollar / SLO window。
- resource-hours saved。
- checkpoint and restore latency。
- fork/branch latency。
- object cache hit rate。
- image/package/cache touched bytes。
- cross-node and cross-region traffic。
- active/idle/wait ratio。
- policy enforcement overhead。
- provenance log overhead。

### Experiments

1. **E1: Workload characterization.**
   展示 production trace 的 session duration、burstiness、artifact size、idle ratio、policy events、object reuse。

2. **E2: Useful start breakdown.**
   分解 scheduling、runtime、image、dependency、object、policy、first action 各阶段耗时，并与 cold start-only metric 对比。

3. **E3: Burst scaling.**
   在上千/上万 agent process fanout 下比较 startup storm、registry/object-store pressure、p99 useful start。

4. **E4: Checkpoint-on-idle.**
   比较 no checkpoint、rootfs-only checkpoint、AKernel full lifecycle 的 resource-hours、restore latency、task completion 和 cost。

5. **E5: Object locality.**
   比较 locality-aware scheduling 与 CPU-only scheduling 的 object hit rate、cross-node traffic、task latency 和 cost。

6. **E6: Branch/fork workflow.**
   评估 parallel exploration、RL rollout、candidate patch evaluation 中 fork/commit/abort 的延迟和状态复用收益。

7. **E7: Policy/provenance overhead.**
   测 eBPF NAT、proxy、secret injection、audit log、provenance span 对 useful start 和 steady-state execution 的影响。

8. **E8: Build Your Own Cluster.**
   从新 region 部署到可运行 Agent workload，测部署时间、人工步骤、失败恢复和跨 region 数据移动。

## 最小证据包

1. 真实 Agent workload trace，覆盖 realm/process/object/tool/policy/resource。
2. 与 AI sandbox 平台的能力对比，证明 sandbox API 已商品化。
3. Time-to-useful-action breakdown，证明 cold start 不够。
4. 端到端 agent-goodput / cost-to-solution 实验。
5. Checkpoint-on-idle / hibernate 资源节省实验。
6. Object locality 实验。
7. Burst-scale startup/restore 实验。
8. Policy/provenance overhead 实验。
9. Build Your Own Cluster 部署实验。

## Related Work 定位边界

### 与 SigmaOS / YuanRong

SigmaOS 和 YuanRong 已经证明 cloud OS / general-purpose serverless kernel 是可行方向。AKernel 不能重复这个大叙事，而要说明 Agent workload 带来了新的 OS 对象：Agent Realm、Agent Process、Agent Object、policy/provenance-carrying lifecycle。

### 与 Ray / workflow systems

Ray、workflow engine 和 serverless DAG 系统擅长 task/actor/DAG 调度，但通常不把 sandbox isolation、untrusted code、workspace checkpoint、credential policy、external side effect 和 provenance 作为同一个 OS lifecycle 管理。

### 与 AI sandbox 平台

E2B、Modal、Daytona、OpenSandbox、Cube、Kubernetes Agent Sandbox 等是最重要的工业 baseline。AKernel 的差异不是“也能启动 sandbox”，而是以真实 trace 驱动的 datacenter-level state/data/policy/control co-design。

### 与 AgentCgroup / Crab / DeltaBox

这些工作证明 agent execution 已经成为 OS 问题。AKernel 应把它们作为强相关工作：AgentCgroup 更关注单 sandbox/session 内 tool-call resource dynamics；Crab/DeltaBox 更关注 checkpoint/rollback；AKernel 的贡献是把 resource、state、object locality、policy 和 deployment 提升到 cloud OS 层。

## 审稿风险与防守策略

1. **风险：被认为是系统集成。**
   防守：用 production trace 证明新的 workload insight，再把 insight 映射到新的 OS semantics 和端到端收益。

2. **风险：与 Kubernetes Agent Sandbox / Cube / OpenSandbox 过近。**
   防守：主动承认 sandbox primitive commodity 化，强调 realm/object/locality/provenance/goodput，而不是 sandbox API。

3. **风险：trace 只来自内部，泛化性不足。**
   防守：覆盖 training、serving、coding、ops、data/browser 多类 workload，并用公开 benchmark 复现实验趋势。

4. **风险：LLM serving 论文认为你忽略模型侧瓶颈。**
   防守：明确 LLM calls treated as external services；本文聚焦 agent execution substrate，可与 LLM serving orthogonal。

5. **风险：安全 claim 过大。**
   防守：给出清晰 threat model，把 isolation、policy enforcement、provenance、human approval 和 external side effect rollback 的边界写清楚。

## 下一步 TODO

1. 把 ACompany 的 training/serving 数据填入 Background 和 Characterization。
2. 补一张 sandbox commodity 能力对比表，注意引用公开资料。
3. 确定 `time-to-useful-action` 在每类 workload 中的精确定义。
4. 明确 AKernel 当前实现已经完成、正在实现、计划实现的边界。
5. 从 trace 中挑 3-5 个最有说服力的 insight，决定 Introduction 的叙事顺序。
6. 给 Evaluation 每个实验绑定具体 baseline、数据集和预期图表。

## 最终定位

AKernel 不是另一个 AI sandbox。它是一个面向 Agent 的云操作系统，把执行、状态、数据、策略、审计和部署统一成 Datacenter Use 的可编程底座。
