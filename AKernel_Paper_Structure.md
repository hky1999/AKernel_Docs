# AKernel 论文结构迭代稿

目标会议：OSDI / SOSP  
暂定题目：**AKernel: An Agent-Native Cloud Operating System for Datacenter Use**

这份文档不是最终论文大纲，而是下一轮讨论用的“结构蓝图”。核心目的有三个：

1. 避免把论文写成 E2B / Modal / OpenSandbox / Cube Sandbox 已经覆盖的 AI sandbox API 复述。
2. 把 Datacenter Use 从 slogan 收敛成可测量、可证伪、可被系统设计支撑的 workload insight。
3. 明确 motivation 章节必须拿到哪些真实 workload 数据，否则论文会缺少顶会级说服力。

## 0. 一句话主张

现有 AI sandbox 平台把 Agent 放进一个“安全 Linux 电脑”；AKernel 要证明的是，Agent 时代需要的是一个 **agent-native cloud OS**：它把 sandbox、状态、artifact、数据局部性、权限、调度和集群部署统一成 Agent 可调用、系统可优化、运维可观测的执行内核。

换句话说，论文不能只说：

> We provide an SDK to create sandboxes.

而应该说：

> We characterize production agent workloads and show that sandbox creation is only one stage in a larger execution lifecycle. Agent workloads are stateful, bursty, artifact-heavy, idle-prone, and policy-constrained. AKernel introduces an Agent Process abstraction and a cross-layer runtime that manages execution, state, data locality, and resource reclamation as first-class OS concerns.

## 1. 需要定义的 AKernel 概念

参考 SigmaOS 的 `proc`、`realm`、`named object` 等抽象，AKernel 也需要自己的概念体系。建议先定义 4 个概念。

### 1.1 Agent Process

Agent Process 是 AKernel 中最小的可调度 Agent 执行单元。它不是传统 FaaS function，也不是长期在线 service，而是一个带状态、带文件系统、带权限、带网络策略、可挂起恢复的云端进程。

Agent Process 包含：

- compute context：CPU / memory / GPU / browser / runtime / image
- execution context：commands, tool calls, running processes, open services
- state context：filesystem diff, memory snapshot, object references, checkpoints
- policy context：network egress, credential capability, data access scope
- provenance context：input prompt/tool call、命令、文件变更、网络访问、产物之间的因果关系

### 1.2 Agent Realm

Agent Realm 是一组协作 Agent Process 的隔离域，类似 SigmaOS realm，但面向 Agent 工作流。一个 Realm 可以对应：

- 一次 coding agent 任务
- 一次 RL rollout / evaluation batch
- 一个多 Agent swarm
- 一个用户会话
- 一个企业任务域

Realm 管理共享数据、权限边界、任务生命周期和资源预算。它让 AKernel 能讨论“一个 Agent 任务整体”的成本、延迟和成功率，而不是只讨论单个 sandbox 的启动延迟。

### 1.3 Agent Object

Agent Object 是 Agent 产生、消费、共享和持久化的统一对象。它可以是：

- 代码仓库快照
- Python package cache
- 浏览器 trace / DOM snapshot
- terminal log
- 中间数据集
- model checkpoint
- sandbox filesystem diff
- Agent checkpoint image

这个概念可以和 openYuanRong Data System 对齐：Object / KV 不只是存储 API，而是 Agent 状态和数据局部性调度的基础。

### 1.4 Datacenter Function

Datacenter Function 是面向 Agent 的调用语义：看起来像一次函数调用，但实际会展开为一个或多个 Agent Process、对象依赖、网络权限、调度决策和状态生命周期。

示意：

```python
run = akernel.submit(
    image="agent-env:latest",
    task="evaluate 10k candidate patches",
    resources={"cpu": 8, "mem": "32Gi"},
    objects=[repo, tests, model_endpoint],
    policy=capabilities,
    lifecycle="checkpoint-on-idle"
)
result = run.await_result()
```

这个 API 本身不是贡献；贡献是它背后的语义：状态可挂起、artifact 可复用、权限可审计、资源可回收、调度可利用 locality。

## 2. Motivation 章节应该怎么写

Motivation 的目标不是“Agent 需要 sandbox”，而是证明：

1. sandbox API 已经 commodity 化；
2. 真实 Agent workload 暴露了 sandbox API 之上的系统问题；
3. 这些问题需要 cloud OS 级别的跨层设计。

### 2.1 Motivation-1: AI sandbox 已经不是稀缺能力

应该先主动承认并量化已有系统：

- E2B 已经提供 sandbox create/connect/kill/pause/auto-resume、命令执行、文件系统、Git、网络规则、模板、快照、metrics 和 BYOC。
- Modal Sandboxes 已经提供 exec、文件、端口/隧道、GPU、Secrets、Volumes、网络 allowlist 和快照。
- Daytona 提供 full computer / dev environment 风格的 SDK、CLI、MCP、process/fs/git/code execution、snapshot、SSH/preview 和多租户控制面。
- OpenSandbox / Cube Sandbox / Kubernetes Agent Sandbox 说明“Agent sandbox runtime + API + snapshot”已经从产品走向开源和云原生标准化。

这一节要形成一个非常重要的转折：

> Therefore, AKernel cannot claim novelty by exposing a sandbox API. The question is what system semantics are missing once sandboxes become abundant.

需要放一张表：

| System | Sandbox lifecycle | FS/process API | Network/policy | Snapshot | BYOC / K8s | Datacenter-level state/data scheduling |
|---|---:|---:|---:|---:|---:|---:|
| E2B | yes | yes | yes | yes | yes | no public semantics |
| Modal | yes | yes | yes | yes | partial | no public semantics |
| Daytona | yes | yes | yes | yes | yes | no public semantics |
| OpenSandbox | yes | yes | runtime-pluggable | yes | yes | limited |
| Cube Sandbox | yes | E2B-compatible | eBPF/security features | yes | runtime focus | limited |
| AKernel | yes | yes | yes | yes | yes | target contribution |

这张表的最后一列必须谨慎写：不是说别人完全没有，而是说公开材料主要停留在 sandbox primitive，没有给出 workload trace 驱动的 datacenter-level state/data/control co-design。

### 2.2 Motivation-2: Agent 不是 FaaS，也不是 microservice

这一节必须靠真实数据。建议采集 AKernel 内部或合作方的 Agent workloads，至少覆盖 3 类：

1. Coding / SWE Agent：修改代码、跑测试、生成 patch。
2. Agentic RL / evaluation：并发 rollout、评测、蒸馏、调用本地或远端推理服务。
3. Data / Web / Ops Agent：数据分析、浏览器自动化、运维诊断、Spark/FaaS 混合任务。

需要采集的 trace 字段：

| 数据项 | 为什么重要 | 预期证明的 insight |
|---|---|---|
| session duration | Agent 任务不是短函数 | 需要 long-lived execution lifecycle |
| active compute time vs idle/wait time | Agent 经常等待 LLM、外部 API、人审、网络 | checkpoint-on-idle 能释放资源 |
| tool-call count per session | Agent 是多阶段交互 | 单次 function invocation 抽象不够 |
| sandbox create/fork/restore count | Agent 会分支、回滚、重试 | snapshot 是 planning primitive |
| artifact count and size | 文件/日志/网页/中间数据是主状态 | 需要 Agent Object，而非无状态函数 |
| cross-step artifact reuse | 状态复用决定成本 | 需要 object locality 和 cache-aware scheduling |
| image size and touched working set | 冷启动瓶颈不等于 VM boot | lazy loading/P2P 要结合 workload |
| network egress domains/API calls | Agent 需要受控访问外界 | policy/capability 是一等语义 |
| failure/retry/human intervention | Agent 执行不确定 | 需要 checkpoint、replay、审计 |
| CPU/memory/GPU/browser timeline | 资源需求随阶段变化 | 需要 agent-aware resource management |

建议图：

- Figure 1：Agent session timeline，标出 LLM thinking、sandbox active、external wait、checkpoint、restore。
- Figure 2：CDF of session durations，与典型 FaaS invocation 对比。
- Figure 3：active/idle ratio，展示大量资源浪费来自 idle-held sandbox。
- Figure 4：artifact size distribution，展示文件/对象状态是主成本。
- Figure 5：image size vs actually touched bytes，证明完整拉镜像不是合理默认。

### 2.3 Motivation-3: 端到端瓶颈不是单点 cold start

AI sandbox 竞品通常会强调 50ms、60ms、100ms 级启动。但对 Agent task 来说，端到端时间可能由多段组成：

```text
T_task =
  T_schedule
+ T_runtime_start
+ T_image_working_set
+ T_dependency_import
+ T_object_fetch
+ T_policy_setup
+ T_tool_execution
+ T_external_wait
+ T_checkpoint_or_restore
+ T_artifact_commit
```

需要用实验拆解一次 Agent Process 启动和一次完整任务：

| 阶段 | 要测什么 | 为什么 |
|---|---|---|
| scheduler decision | p50/p99 scheduling latency | 证明控制面不是瓶颈或定位瓶颈 |
| sandbox boot | runtime start latency | 和 AFaaS/OpenSandbox/Cube/K8s 对比 |
| image working set | touched bytes, fetch latency | 支撑 lazy loading / P2P |
| dependency import | Python/Node/browser warmup | Agent 环境经常重依赖 |
| object restore | checkpoint/object load latency | 支撑 Data System 改造 |
| network/policy setup | eBPF/NAT/rules overhead | 支撑安全访问外界 |
| first useful action | time-to-first-command / first token-independent action | 比 cold start 更接近用户体验 |

这一节的结论应该是：

> Optimizing VM/container boot alone is insufficient. Agent execution needs cross-layer scheduling over runtime state, image blocks, object locality, and policy setup.

### 2.4 Motivation-4: 状态生命周期是 Agent 的核心 OS 问题

Agent 的状态不是只有 filesystem snapshot。至少有 5 层：

1. image/base environment
2. dependency/package cache
3. workspace/filesystem diff
4. in-memory process state
5. external object references and provenance log

需要采集：

- checkpoint size 分布
- restore 后实际访问的 working set
- checkpoint 频率和触发原因：idle、branch、failure recovery、human review、migration
- checkpoint/restore 对任务成功率、恢复时间、成本的影响
- 分支执行中重复状态的比例

建议实验：

- no checkpoint：sandbox 长时间占资源
- rootfs-only checkpoint：释放部分资源，但恢复慢或状态不完整
- AKernel full lifecycle：idle checkpoint + object store + locality-aware restore

指标：

- resource-hours saved
- restore latency
- task completion time
- successful task cost
- lost work after failure

### 2.5 Motivation-5: Data locality 比传统 sandbox API 更重要

Agent 会产生很多中间对象，而这些对象会被后续 tool calls 反复访问。AKernel 可以把 openYuanRong Data System 作为关键差异点，但必须用数据说话。

需要采集：

- object read/write size distribution
- objects per session
- reuse distance：一个 object 多久后再次被访问
- fanout/fanin：一个 object 被多少 Agent Process 共享
- same-node vs cross-node data transfer latency
- object locality 命中率对端到端任务的影响

可写成一个 insight：

> For agent workloads, the scheduler should place computation near state, not merely place containers near free CPU.

这会把 AKernel 和 E2B 类产品拉开：E2B 的抽象更像“给 Agent 一台电脑”，AKernel 的抽象是“给 Agent 一个可调度、可迁移、可共享对象的数据中心进程”。

### 2.6 Motivation-6: 成功率/成本/延迟要联合优化

系统论文不能只报 throughput。Agent workloads 更自然的目标是：

```text
Agent Goodput = successful tasks / dollar / SLO window
```

或者：

```text
Cost-to-solution = total cloud cost / completed correct tasks
```

需要采集：

- 每个任务的 end-to-end cost
- 成功/失败/重试次数
- 人工介入次数
- SLO violation
- sandbox/runtime overhead 对最终成功率的影响

这一点可以成为 AKernel 的 evaluation 主线：AKernel 不是单点更快，而是在同等成功率下更便宜，或在同等成本下更快完成更多正确任务。

## 3. 论文贡献应该怎么定

建议贡献写成 4 条，但真正强贡献最好控制在 3 条。

### Contribution 1: Production characterization of agent workloads

这是最关键的贡献。如果没有这条，AKernel 很容易变成“系统集成工程”。

要回答：

- Agent workload 和 FaaS/microservice 有什么结构性不同？
- 为什么现有 sandbox API 不够？
- 哪些 trace 事实直接驱动 AKernel 设计？

至少要有一节 workload characterization，类似 Azure Functions trace / Borg / DL cluster traces 的写法。

### Contribution 2: Agent Process abstraction

提出 Agent Process / Agent Realm / Agent Object / Datacenter Function。

重点不是 API，而是语义：

- stateful lifecycle：create, fork, checkpoint, restore, hibernate, migrate, commit
- object-aware execution：workspace 和 object store 统一
- policy-carrying execution：每个 process 带 capability
- provenance：每个 artifact 能追溯到 prompt/tool/command/network event

### Contribution 3: Cross-layer implementation in AKernel

AKernel 基于 openYuanRong + AFaaS + Data System + lazy image + P2P + eBPF NAT + Terraform/Helm 实现上述抽象。

这一节要小心：不要把 openYuanRong、AFaaS、Dragonfly、Nydus 各自已有的能力包装成 AKernel 原创。AKernel 的原创点应是跨层连接：

- scheduler 利用 image/object/checkpoint locality
- C/R 与 Data System 合并，而不是孤立 snapshot
- Agent Process 的 policy/provenance 贯穿 gateway、scheduler、sandbox daemon、data worker
- deployment 和 observability 服务于 Agent self-operation

### Contribution 4: End-to-end evaluation on real agent workloads

必须使用真实或半真实 workload，而不是只用 microbenchmark。

建议 benchmark suite：

| Workload | 来源/构造 | AKernel 要证明什么 |
|---|---|---|
| Coding agent | SWE-bench 或内部 coding tasks | workspace/artifact/state 重要 |
| Terminal/Ops agent | Terminal-Bench 或内部运维任务 | shell sandbox 和 long-running state 重要 |
| Agentic RL rollout | 内部 RL/eval workload | burst scale 和 object locality 重要 |
| Data analysis agent | Python/Spark/FaaS 混合任务 | Datacenter Function 比单 sandbox 更自然 |
| Web/enterprise agent | WebArena/WorkArena 类任务 | browser/network/policy/provenance 重要 |

## 4. 推荐论文大纲

### 1. Introduction

叙事顺序：

1. Computer Use 给 Agent 一个虚拟电脑。
2. 但下一阶段 Agent 需要 Datacenter Use：并发、数据、状态、隔离、成本。
3. AI sandbox API 已经快速商品化，不能作为论文创新点。
4. 我们通过生产 trace 发现 Agent workloads 具有 5 个特征：stateful, bursty, artifact-heavy, idle-prone, policy-constrained。
5. AKernel 提出 Agent Process 抽象，并用 cloud OS 方式跨层管理执行、状态、数据和权限。
6. 总结贡献。

### 2. Background and Related Systems

不要写成大杂烩。建议按“为什么不够”组织。

2.1 Cloud OS and distributed execution  
SigmaOS, YuanRong, Ray, Borg/Omega/SkyPilot。

2.2 Serverless and stateful functions  
Cloudburst, Beldi, Kappa, Faasm, Pocket, SONIC, gg, PyWren/Lithops。

2.3 Sandboxes and cold start  
AFaaS, Firecracker, SEUSS, Catalyzer, FaaSnap, SOCK, OpenSandbox, Cube Sandbox。

2.4 AI sandbox products  
E2B, Modal, Daytona, Morph, OpenAI Agents Sandbox, Kubernetes Agent Sandbox。

章节结论：

> Prior systems optimize either cloud execution, serverless state, cold start, or sandbox APIs. AKernel targets the intersection exposed by agent workloads: stateful, policy-carrying, artifact-heavy execution at datacenter scale.

### 3. Agent Workload Characterization

这是 motivation 的主体。建议结构：

3.1 Methodology: traces, privacy, workloads, instrumentation  
3.2 Agents are long-lived and idle-prone  
3.3 Agents are artifact-heavy and reuse state  
3.4 Agents are bursty and branchy  
3.5 Agents are policy-constrained  
3.6 Implications for system design

最后列出 design requirements：

- R1: sub-second useful start under large images and heavy dependencies
- R2: cheap hibernation and fast restore
- R3: object-local scheduling
- R4: capability-carrying execution
- R5: causal observability and provenance
- R6: agent-goodput optimization

### 4. AKernel Abstractions

4.1 Agent Process  
4.2 Agent Realm  
4.3 Agent Object  
4.4 Datacenter Function  
4.5 Lifecycle semantics

生命周期状态机建议：

```text
Created -> Starting -> Running -> Idle -> Checkpointed
    |          |          |        |          |
    |          |          v        v          v
    |          |       Forked   Hibernated  Restoring
    |          |          |        |          |
    +----------+----------+--------+----------+
                           |
                        Committed / Failed
```

### 5. System Design

5.1 Control plane: SDK/CLI/Gateway/Scheduler  
5.2 Node agent: Proxy/Sandbox Daemon/AFaaS/NanoVisor  
5.3 Data plane: Data Worker/Object/KV/shared memory  
5.4 Image plane: lazy loading/P2P/cache affinity  
5.5 State plane: checkpoint/restore/object-backed snapshots  
5.6 Policy and network plane: eBPF NAT/capabilities/audit  
5.7 Deployment plane: Build Your Own Cluster as reproducible infrastructure

### 6. Implementation

写具体但克制：

- openYuanRong 函数系统如何改造以支持 Agent Process
- Data System 如何存 Agent Object / checkpoint
- AFaaS 如何集成 sandbox daemon
- distill-fs / Dragonfly / lazy loading 如何接入
- eBPF NAT 和双向代理如何实现
- Terraform + Helm 多云部署
- observability：trace/log/metrics 如何按 Agent Realm 聚合

### 7. Evaluation

#### 7.1 Workload characterization results

先用 trace 支撑 motivation，不然系统实验会显得任意。

#### 7.2 End-to-end task goodput

对比：

- AKernel
- Kubernetes + containerd/gVisor
- openYuanRong baseline
- E2B-like single sandbox baseline，如果无法直接测 E2B，可实现等价 API baseline
- AKernel minus C/R
- AKernel minus locality scheduling
- AKernel minus lazy image/P2P

指标：

- successful tasks/hour
- cost per successful task
- time-to-solution
- p50/p95/p99 task latency
- SLO violation

#### 7.3 Lifecycle efficiency

测 create/fork/checkpoint/restore/hibernate/migrate。

指标：

- latency
- checkpoint size
- restore working set
- resource-hours saved
- failure recovery time

#### 7.4 Data locality and object sharing

测 object-aware scheduling。

指标：

- object cache hit rate
- cross-node traffic
- data transfer latency
- task runtime improvement

#### 7.5 Burst scale

测 1、100、1k、10k Agent Process 并发创建/恢复。

注意：这里不只报 cold start，要报 time-to-first-useful-action 和 time-to-all-ready。

#### 7.6 Policy/provenance overhead

测 capability/network/audit 的开销和收益。

指标：

- policy setup latency
- egress enforcement overhead
- audit log size
- blocked unauthorized access cases
- replay/debug time reduction

#### 7.7 Build Your Own Cluster

把文章里的 8 分 39 秒部署做成系统实验：

- 多云多区域重复部署 N 次
- time-to-ready 分解：VPC、K8s、node pool、Helm、health check
- failure recovery
- manual steps
- deploy-to-first-Agent-Process

这一节适合放在 evaluation 后半或 experience。

### 8. Production Experience

可写但不要当核心贡献：

- 三人维护经验
- AI-assisted debugging case study
- 内部落地 workload
- 哪些组件最容易出问题
- 用户 API 演进
- 运维成本变化

### 9. Discussion and Limitations

必须诚实：

- GPU/NPU 支持还不完整
- Windows/macOS/Android sandbox 是未来工作
- deterministic replay 可能只能做到 best-effort
- E2B/Modal 等闭源服务无法完全公平对比
- 安全证明如果没有形式化，只能说 capability enforcement，不要过度声称

## 5. Related Work 应该怎么定位

### 5.1 SigmaOS

可借鉴：

- cloud OS 叙事
- proc/realm/object 风格抽象
- 用统一操作系统接口覆盖 serverless 和 microservice

AKernel 差异：

- SigmaOS 解决 serverless 和 microservice 的统一；
- AKernel 解决 Agent execution 的状态、artifact、权限和 datacenter lifecycle。

### 5.2 YuanRong / openYuanRong

可借鉴：

- 分布式内核
- 函数系统 + Data System
- 多语言 runtime 和生产级 serverless

AKernel 差异：

- YuanRong 是通用 serverless/distributed application substrate；
- AKernel 在其上定义 Agent Process，并改造状态、对象、调度和生命周期语义。

### 5.3 AFaaS

可借鉴：

- serverless/AI agent 冷启动路径
- sandbox clone/fork
- gVisor/nanovisor、distill-fs、P2P 镜像

AKernel 差异：

- AFaaS 是节点侧 runtime/cold-start 贡献；
- AKernel 是集群级 Agent OS，冷启动只是 Agent lifecycle 的一段。

### 5.4 E2B / Modal / Daytona / Morph

这些系统是最危险的相关工作，因为它们已经覆盖了很多 API。

AKernel 不能说：

- 我们提供 sandbox SDK。
- 我们支持 exec / file / network / snapshot。
- 我们能给 Agent 一个 Linux 环境。

AKernel 应说：

- 我们用 production trace 定义 Agent Process。
- 我们把 state lifecycle、object locality、policy 和 provenance 放进调度与数据平面。
- 我们在 datacenter scale 下优化 agent-goodput，而非单 sandbox latency。

### 5.5 OpenSandbox / Cube Sandbox / Kubernetes Agent Sandbox

这些说明“AI sandbox runtime 标准化”正在发生。

AKernel 差异：

- 不和它们比“谁的 VM 启动更快”作为主线；
- 把 runtime 当 substrate；
- 把贡献放在 Agent Realm / Agent Object / lifecycle / locality / goodput。

## 6. 最小可投稿证据包

如果时间有限，建议至少拿到下面这些证据：

1. 一份真实 Agent workload trace，至少 1k sessions，最好覆盖 3 类 workload。
2. 一张 AI sandbox 商品化功能对比表。
3. 一组端到端 task-goodput 实验，证明 AKernel 优于 K8s/openYuanRong baseline。
4. 一组 checkpoint/restore 资源节省实验。
5. 一组 object locality 实验。
6. 一组 burst scale 实验。
7. 一个 production experience case study。

没有第 1 条，论文会像系统集成；没有第 3-5 条，论文会像概念宣言。

## 7. 下一轮需要向 AKernel 作者确认的问题

1. 是否能拿到生产 trace？能否脱敏公开统计分布？
2. 现有 AKernel 是否已经有 Agent Realm / Object / Process 的内部概念，还是需要论文侧重新抽象？
3. C/R 当前保存了哪些状态：rootfs、memory、process、network connection、object refs？
4. openYuanRong Data System 是否能暴露 object locality / cache hit / cross-node traffic 指标？
5. 调度器目前是否能感知 image cache、checkpoint location、object location？
6. eBPF NAT / 双向代理 / policy 是否有 per-Agent 规则和审计日志？
7. Agentic RL、Spark、Serverless Function 三类内部 workload 中，哪一类最能展示 AKernel 独特性？
8. 是否愿意把 Build Your Own Cluster 放在主线，还是作为 experience？
9. 是否能对比 E2B/Modal/OpenSandbox/Cube，还是只能做本地等价 baseline？
10. 三人维护的数据能否量化：on-call 事件、部署频率、故障定位时间、代码仓库规模、AI-assisted debugging 案例。

## 8. 当前建议的论文主线版本

最稳的版本：

> We present AKernel, an agent-native cloud operating system. Based on production traces, we show that agent workloads are stateful, bursty, artifact-heavy, idle-prone, and policy-constrained. Existing sandbox platforms expose useful execution primitives, but leave state lifecycle, data locality, policy, and agent-level goodput to applications. AKernel introduces Agent Processes, Agent Realms, and Agent Objects, and implements them with a cross-layer runtime built on openYuanRong and AFaaS. On real agent workloads, AKernel improves cost-to-solution and time-to-useful-action by co-designing scheduling, checkpoint/restore, object storage, image loading, and policy enforcement.

这个版本的优点是：

- 承认 sandbox API 已有，不硬抢；
- 把论文贡献锚定在 trace + abstraction + cross-layer system；
- 能自然继承 SigmaOS/YuanRong/AFaaS；
- evaluation 有明确闭环。

## Appendix A. Trace 埋点草案

为了让 motivation 不停留在“我们觉得 Agent 是这样”，建议从现在开始按 Agent Realm 维度采集 trace。可以先不追求完美，先拿到能画 CDF 和做阶段分解的数据。

### A.1 Realm-level fields

```json
{
  "realm_id": "uuid",
  "workload_type": "coding|rl_eval|data|web|ops|spark|faas",
  "user_or_team": "anonymized",
  "start_ts": 0,
  "end_ts": 0,
  "status": "success|failed|timeout|cancelled|human_intervention",
  "success_metric": "tests_passed|reward|answer_correct|job_completed",
  "total_cost": 0.0,
  "human_interventions": 0,
  "slo_ms": 0
}
```

### A.2 Process-level fields

```json
{
  "process_id": "uuid",
  "realm_id": "uuid",
  "parent_process_id": "uuid|null",
  "lifecycle_event": "create|start|fork|idle|checkpoint|restore|hibernate|migrate|commit|fail",
  "runtime": "runc|gvisor|nanovisor|kata|firecracker",
  "image": "hash",
  "requested_resources": {"cpu": 0, "mem_mb": 0, "gpu": 0},
  "actual_usage_timeseries": "metrics_ref",
  "node_id": "anonymized",
  "latency_ms": 0
}
```

### A.3 Object-level fields

```json
{
  "object_id": "uuid",
  "realm_id": "uuid",
  "producer_process_id": "uuid",
  "type": "repo|file|log|checkpoint|dataset|model|browser_trace|package_cache",
  "size_bytes": 0,
  "read_count": 0,
  "write_count": 0,
  "reuse_distance_ms": 0,
  "node_location": "anonymized",
  "cache_hit": true
}
```

### A.4 Tool/policy/provenance fields

```json
{
  "event_id": "uuid",
  "realm_id": "uuid",
  "process_id": "uuid",
  "event_type": "llm_call|tool_call|shell|file_write|network|credential|policy_block",
  "tool_name": "bash|python|browser|spark|faas|custom",
  "input_size": 0,
  "output_size": 0,
  "duration_ms": 0,
  "egress_domain": "example.com",
  "policy_id": "uuid",
  "allowed": true,
  "artifact_refs": ["object_id"]
}
```

这些字段能直接支撑：

- session duration CDF
- active vs idle ratio
- tool-call fanout
- checkpoint size / restore latency
- object reuse / locality
- unauthorized egress / policy overhead
- cost per successful task

## Appendix B. 调研参考资料

### B.1 AI sandbox / Agent execution platforms

- E2B Docs: https://e2b.dev/docs
- E2B Python SDK Sandbox reference: https://e2b.dev/docs/sdk-reference/python-sdk/v2.15.2/sandbox_sync
- Modal Sandboxes: https://modal.com/docs/guide/sandboxes
- Daytona Docs: https://www.daytona.io/docs/en/
- Morph Cloud Docs: https://cloud.morph.so/docs/developers
- Kubernetes Agent Sandbox blog: https://kubernetes.io/blog/2026/03/20/running-agents-on-kubernetes-with-agent-sandbox/
- OpenAI Code Interpreter guide: https://developers.openai.com/api/docs/guides/tools-code-interpreter
- OpenAI Agents Sandbox guide: https://developers.openai.com/api/docs/guides/agents/sandboxes

### B.2 Sandbox runtime / cold start / image distribution

- OpenSandbox GitHub: https://github.com/opensandbox-group/OpenSandbox
- OpenSandbox secure runtime OSEP: https://open-sandbox.ai/oseps/0004-secure-container-runtime
- Cube Sandbox GitHub: https://github.com/TencentCloud/CubeSandbox
- gVisor Docs: https://gvisor.dev/docs/
- Kata Containers: https://katacontainers.io/
- Firecracker specification: https://github.com/firecracker-microvm/firecracker/blob/main/SPECIFICATION.md
- Firecracker NSDI paper: https://www.usenix.org/conference/nsdi20/presentation/agache
- Nydus snapshotter: https://github.com/containerd/nydus-snapshotter
- Dragonfly: https://d7y.io/
- Catalyzer: https://ipads.se.sjtu.edu.cn/pub/projects/catalyzer
- SEUSS: https://www.cs.bu.edu/~jappavoo/Resources/Papers/seuss.pdf
- FaaSnap: https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf
- SOCK: https://www.usenix.org/conference/atc18/presentation/oakes

### B.3 Cloud OS / serverless / distributed execution

- SigmaOS paper: https://pdos.csail.mit.edu/papers/sigmaos%3Asosp24.pdf
- YuanRong paper: local `References/Chen 等 - 2024 - YuanRong A Production General-purpose Serverless System for Distributed Applications in the Cloud.pdf`
- AFaaS paper: local `References/Chai et al_2025_Fork in the Road.pdf`
- Toward a Cloud OS: https://www.moritzsteiner.de/papers/CloudOS.pdf
- Borg: https://research.google.com/pubs/pub43438.html
- Omega: https://research.google/pubs/omega-flexible-scalable-schedulers-for-large-compute-clusters/
- SkyPilot: https://www.usenix.org/conference/nsdi23/presentation/yang-zongheng
- Serverless in the Wild: https://www.usenix.org/conference/atc20/presentation/shahrad
- One Step Forward, Two Steps Back: https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf
- Cloudburst: https://www.vldb.org/pvldb/vol13/p2438-sreekanti.pdf
- Beldi: https://www.usenix.org/system/files/osdi20-zhang_haoran.pdf
- Kappa: https://people.eecs.berkeley.edu/~zhangwen/papers/kappa.pdf
- Faasm: https://www.usenix.org/system/files/atc20-shillaker.pdf
- SAND: https://www.usenix.org/conference/atc18/presentation/akkus
- Pocket: https://www.usenix.org/conference/osdi18/presentation/klimovic
- SONIC: https://www.usenix.org/conference/atc21/presentation/mahgoub
- PyWren: https://arxiv.org/abs/1702.04024
- gg: https://www.usenix.org/conference/atc19/presentation/fouladi
- Ray: https://arxiv.org/abs/1712.05889

### B.4 Agent workload / benchmarks

- Agentic AI Workload Characteristics: https://arxiv.org/abs/2605.26297
- AgentBench: https://proceedings.iclr.cc/paper_files/paper/2024/hash/e9df36b21ff4ee211a8b71ee8b7e9f57-Abstract-Conference.html
- WebArena: https://proceedings.iclr.cc/paper_files/paper/2024/hash/4410c0711e9154a7a2d26f9b3816d1ef-Abstract-Conference.html
- WorkArena: https://proceedings.mlr.press/v235/drouin24a.html
- OSWorld: https://papers.nips.cc/paper_files/paper/2024/hash/5d413e48f84dc61244b6be550f1cd8f5-Abstract-Datasets_and_Benchmarks_Track.html
- SWE-bench: https://proceedings.iclr.cc/paper_files/paper/2024/hash/edac78c3e300629acfe6cbe9ca88fb84-Abstract-Conference.html
- τ-bench: https://openreview.net/forum?id=roNSXZpUDN
- Terminal-Bench: https://arxiv.org/abs/2601.11868
- AIOS: https://arxiv.org/abs/2403.16971
