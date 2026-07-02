# Agent 云原生调度与弹性执行专项 Survey

时间：2026-07-02T10:07:31Z  
主题：面向 AKernel 论文的三条核心系统问题：分布式调度、水平扩容/冷启动、单实例垂直弹性。  
调研方式：在主线检索之外，按项目要求启用 3 个并行 subagent，分别覆盖 distributed scheduling、horizontal scaling/useful start、vertical scaling/resource dynamics。  
边界：本文**不讨论 LLM serving / inference / model placement / KV cache / batching**。只讨论如何给 agent 提供沙箱环境、状态环境、对象/文件环境、安全策略和云原生编排环境。

## 0. Executive Summary

上一版 survey 已经证明：agent workload 正在从 benchmark 走向 OS/resource/sandbox/workflow/productization。本文进一步收窄到 AKernel 最应该切入的系统空间：

> 现有云原生系统能调度 Pod / Job / Function，现有 sandbox 平台能启动隔离环境，现有 autoscaler 能调整 replica 或 container request，但它们缺少一个共同抽象：把 agent 的执行状态、工作区对象、checkpoint lineage、网络/凭证策略、工具调用阶段、资源突发和审计 provenance 绑定成一个可调度、可恢复、可弹性管理的 cloud OS object。

对应到 AKernel，可以把论文主张写成：

> AKernel is a cloud operating system for agent execution. It schedules **Agent Realms**, places and hydrates **Agent Objects**, and elastically controls **Agent Processes** so that stateful, bursty, policy-carrying agent workloads can scale across datacenter resources.

本轮最重要的三个结论：

1. **云原生 agent 需要 realm-aware distributed scheduling**  
   Kubernetes、serverless orchestrator、workflow engine、batch scheduler 的单位分别是 Pod、function、DAG task、job。Agent workload 的真正单位是 `Agent Realm`：一个会话/任务/rollout/workflow 的隔离域、状态域、策略域和对象局部性域。调度器必须同时考虑 image cache、workspace/checkpoint location、object reuse、credential/network policy、side-effect provenance 和 high-churn control path。

2. **高并发水平扩容不能只测 cold start，要测 useful start**  
   FaaS 和 microVM 论文已经把 runtime startup 优化到 ms 级甚至亚 ms 级；agent 的用户可感知延迟却是 `request accepted -> first useful action completes`。这条路径包含调度、鉴权、网络策略、sandbox restore、workspace/object hydration、repo/package/browser/tool server ready、首条命令或工具调用成功。AKernel 应把 `time-to-useful-action` 作为核心 metric，而不是只报 container start 或 shell ready。

3. **Agent 单实例需要 phase-aware vertical elasticity**  
   AgentCgroup 已经证明 sandboxed coding agents 的资源需求与 tool-call phase 强相关，container-level policy mismatch 明显。传统 VPA/in-place resize 仍然以 Pod/container 历史资源为中心，不能表达 build/test/browser/IO/wait/checkpoint/retry 等 agent phase。AKernel 应把 `Agent Process` 定义为 process tree + workspace + tool phase + resource contract，并提供 fast cgroup/eBPF control + slow realm scheduler 的分层控制。

最适合 AKernel 的论文 gap：

| 方向 | 现有系统做到什么 | 仍缺什么 | AKernel 可声明的贡献 |
|---|---|---|---|
| Distributed scheduling | K8s 调 Pod，Kueue 调 batch queue，Dirigent/KubeDirect 优化 FaaS control path，workflow systems 调 DAG | 不理解 agent realm、workspace/checkpoint/object locality、policy/provenance | Realm-aware scheduler + object locality + policy-carrying placement |
| Horizontal scaling | Firecracker/SEUSS/FaaSnap/Catalyzer/AFaaS/lazy image 优化 runtime cold start | 不测 agent useful start；不把 repo/tool/cache/secret/network policy 纳入关键路径 | Useful-start-optimized scaling pipeline |
| Vertical scaling | AgentCgroup/cgroup/PSI/VPA 能观测或调整资源 | 粒度仍是 host/container/tool-call，缺 realm/object/provenance 语义 | Phase-aware vertical elasticity for Agent Processes |

## 1. 论文问题应如何重新表述

### 1.1 不要把 AKernel 写成 sandbox platform

E2B、Modal、Daytona、Cloudflare Sandboxes、GKE Agent Sandbox、Docker Sandboxes、OpenSandbox、Cube Sandbox 等已经把“创建一个可执行 agent code 的 sandbox”变成商品化能力。AKernel 如果只说“我们提供 sandbox SDK / remote shell / file API / snapshot”，会被工业系统覆盖。

更强的表述是：

> Sandbox API is becoming a commodity. The missing operating-system problem is how to schedule, hydrate, resize, recover, and audit thousands of stateful agent execution domains under bursty demand and policy constraints.

### 1.2 不要把 AKernel 写成 LLM serving system

Parrot、AgentServe、Continuum、Helium、Kairos、Agentix/Autellix、Murakkab、Cortex 等研究非常重要，但它们主要优化 LLM-side execution、workflow serving 或 model-side cache/scheduling。AKernel 本文应明确排除这条线：

> We treat LLM calls as external services. AKernel optimizes the execution substrate around agents: sandbox placement, state restoration, object locality, resource elasticity, and policy/provenance.

### 1.3 建议的三条 problem statement

**P1. Realm-aware scheduling gap**

Cloud schedulers place pods, functions, or jobs. Agent workloads require scheduling an execution realm: multiple related sandboxes/processes, shared objects, checkpoints, secrets, network policy, audit lineage, and sometimes human/external waits. Existing schedulers do not expose these semantics.

**P2. Useful-start scaling gap**

Serverless cold-start work optimizes boot/restore/runtime initialization. For agents, the first useful action depends on whether the workspace, dependencies, browser/tool servers, policy, credentials, and object cache are ready. A sandbox that has booted but cannot run `git status`, a test command, or a browser action is not useful.

**P3. Phase-aware vertical elasticity gap**

Agent resource demand changes at tool-call boundaries and inside process trees. Traditional VPA/autoscaling uses coarse historical metrics, while agent execution requires resource leases tied to active/waiting/checkpointable/pinned phases, plus correctness constraints from workspace state and side effects.

## 2. 云原生 Agent 需要的分布式调度

### 2.1 调度对象：从 Pod/Function/Job 到 Agent Realm

AKernel 的 `Agent Realm` 可以被定义为：

> 一个 agent 任务、用户会话、multi-agent workflow 或 RL rollout 的隔离域和资源域，包含一组 Agent Processes、Agent Objects、checkpoint lineage、network/credential policy、provenance log 和 SLO/cost budget。

这个定义比 Kubernetes Namespace、Pod、Job、Ray actor、Dapr actor、FaaS function 都更贴近 agent：

- 它是**状态域**：repo、浏览器 profile、checkpoint、package cache、日志、测试产物都属于 realm。
- 它是**策略域**：credential、egress、filesystem access、secret mount、human approval 属于 realm。
- 它是**局部性域**：计算应靠近 workspace/checkpoint/object/cache，而不是只靠近 CPU 空闲节点。
- 它是**审计域**：agent action 的输入、输出、外部 side effect 和 artifact lineage 都属于 realm。

### 2.2 关键相关工作

| 年份 | 来源 | 系统/机制 | URL | 核心机制/数据 | 对 AKernel 的启发与不足 |
|---|---|---|---|---|---|
| 2024 | SOSP / arXiv | Dirigent | https://arxiv.org/abs/2404.16393 | clean-slate FaaS orchestrator；论文指出 sandbox 初始化可在 10-100ms，但 cluster scheduling latency 可能高几个数量级；报告低延迟创建 2500 sandboxes/s，并显著优于 Knative | 证明高 churn sandbox 的 control path 会成为瓶颈。AKernel 可借鉴 fast path，但不能牺牲 agent provenance/policy。 |
| 2026 | NSDI | KubeDirect | https://www.usenix.org/system/files/nsdi26-qi.pdf | 保留 Kubernetes 生态，但绕过 API Server 做高频状态传递；报告 Azure trace 中可出现 16K instance creations/min，相比 Knative serving latency 26.7x 改善 | 说明 K8s 兼容与高 churn 性能存在张力。AKernel 可采用“control-plane audit + execution fast path”二层设计。 |
| 2026 | Kubernetes SIG | Kubernetes Agent Sandbox | https://agent-sandbox.sigs.k8s.io/docs/ | Kubernetes-native CRD：Sandbox、SandboxTemplate、SandboxClaim、SandboxWarmPool；支持生命周期、暂停/恢复、RBAC、Namespace、NetworkPolicy、Quota | 直接说明 agent sandbox 正在 K8s 对象化；不足是偏 sandbox lifecycle，不提供 realm/object/provenance 一体语义。 |
| 2024 | OSDI / local PDF | SigmaOS | https://dspace.mit.edu/entities/publication/c6420e9f-2469-49ce-baba-9864b9f9175e | 将 serverless 与 microservice workload 统一到 cloud OS 抽象中 | AKernel 可沿用 cloud OS 叙事，但把对象从 service/process 扩展到 agent realm/process/object/policy。 |
| 2024 | local PDF / openYuanRong | YuanRong | https://github.com/open-yuanrong/openyuanrong | 生产级 general-purpose serverless system for distributed applications | 是 AKernel data/execution substrate 的重要参考；不足是没有 agent-native resource/policy/provenance 语义。 |
| 2020 | VLDB / arXiv | Cloudburst | https://arxiv.org/abs/2001.04592 | stateful functions with low-latency data access；将 data locality 带入 serverless scheduling | 支撑 Agent Object locality：agent 不是 stateless function，artifact/cache/checkpoint placement 影响端到端性能。 |
| 2022 | OSDI | ORION | https://www.usenix.org/conference/osdi22/presentation/mahgoub | 对 serverless DAG 做 right-sizing、bundling、prewarming，优化端到端 latency CDF | Agent plan/workflow 可以借鉴 DAG 优化，但 agent 运行时会动态分支，不能完全依赖离线 profiling。 |
| 2023 | OSDI | ExoFlow | https://www.usenix.org/conference/osdi23/presentation/zhuang | 解耦 workflow execution 与 recovery；用 task annotation 表达 nondeterminism，提供 exactly-once 语义 | 对 agent provenance 很关键：外部 side effect、不可重放工具调用、checkpoint policy 应显式进入 AKernel。 |
| 2025 | EuroSys | AlloyStack | https://madsys.cs.tsinghua.edu.cn/publication/alloystack-a-library-operating-system-for-serverless-workflow-applications/eurosys25-you.pdf | serverless workflow LibOS；同一 workflow 内按需加载 OS 组件和引用传递；报告 cold start 降低 98.5% 到 1.3ms | 启发 realm 内强共享/弱隔离；但 arbitrary untrusted agent 需要更强安全边界。 |
| 2022 | USENIX ATC | RunD | https://www.usenix.org/conference/atc22/presentation/li-zijun-rund | 高密度安全容器 runtime；报告单节点每秒启动 200+ 安全容器、384GB 节点部署 2500+ 安全容器 | 说明节点内 runtime density 是 realm scheduling 的下层瓶颈。 |
| 2022 | arXiv | KubeAdaptor | https://arxiv.org/abs/2207.01222 | 解决 workflow scheduler 的任务顺序与 Kubernetes pod 调度顺序不一致 | 说明 workflow intent 不能简单翻译成 Pods。Agent Realm 需要 bridge intent 与 placement。 |
| 2024 | arXiv | HyperFlow cloud-native workflow | https://arxiv.org/abs/2408.15445 | 比较 K8s Job、task clustering、autoscalable worker-pool 等 workflow 模式 | Agent Realm 可类似 resident worker pool，但需要对象/策略/审计语义。 |
| 2026 | Kubernetes | Kueue | https://kueue.sigs.k8s.io/docs/concepts/cluster_queue/ | ClusterQueue、ResourceFlavor、cohort/fair sharing，用于 batch/job admission | 可作为 realm admission/fairness baseline；不足是 quota/job 粒度，不懂 agent object locality。 |
| 2026 | Ray/KubeRay docs | Ray label-based scheduling | https://docs.ray.io/en/master/cluster/kubernetes/user-guides/label-based-scheduling.html | Ray task/actor/placement group 通过 label selector 约束 node placement | 可借鉴 actor/object locality API；不足是 policy/provenance 不是核心。 |
| 2026 | Dapr docs | Dapr Actors / Workflow | https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/ | virtual actor identity、activation、state、placement；workflow 构建在 actors 上 | Agent Object 可借鉴 identity/state/activation；但 Dapr 不提供强 sandbox 与 process-tree resource control。 |
| 2026 | OpenAI Docs | Agents Sandboxes | https://developers.openai.com/api/docs/guides/agents/sandboxes | 将 agent harness 与 sandbox compute 分离；sandbox 暴露文件、命令、端口、隔离和 manifest | 强化“orchestration 与 execution boundary 分离”；但不是公开底层 scheduler。 |
| 2026 | Anthropic Docs | Self-hosted sandboxes | https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes | 控制面与用户自托管工具执行环境分离；worker 从队列 claim work item | 说明 agent 执行环境可跨云/自托管；但队列模型较粗，缺 locality-aware scheduling。 |
| 2026 | Google Cloud Docs | Code Execution Sandbox | https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sandbox/code-execution-overview | 托管安全 sandbox，支持创建、列举、执行代码、保留状态、TTL | 证明 sandbox 是云平台一等能力；不足是平台私有、策略/调度透明度有限。 |

### 2.3 分布式调度的四个 AKernel 设计点

**D1. Two-level scheduling**

AKernel 不应把每个 tool call 都交给 Kubernetes API Server。更合理的设计是：

- `Realm scheduler` 做 admission、placement、fairness、object locality、policy compatibility。
- `Node agent / sandbox daemon` 做高频 start/fork/restore/hibernate/cgroup update。
- `Control plane` 异步持久化审计状态和 provenance，而不是阻塞每个 hot action。

这与 Dirigent/KubeDirect 的 insight 一致，但 AKernel 要保留 Kubernetes 生态和审计语义。

**D2. Object-aware placement**

Agent Object 应至少包含这些调度字段：

| 字段 | 例子 | 调度意义 |
|---|---|---|
| object type | repo, checkpoint, browser profile, package cache, dataset, log, diff | 决定 restore/hydration 策略 |
| location | node, rack, region, object store, remote cache | 决定 compute-to-data 或 data-to-compute |
| reuse distance | same realm, same user, same task family, global cache | 决定 warm pool/cache 保留 |
| confidentiality | public, tenant-private, credential-bearing | 决定可共置范围 |
| mutability | immutable base, COW diff, append-only log, external side effect | 决定 fork/commit/rollback |
| hydration SLA | must be ready before first command / lazy / background | 决定 useful start 关键路径 |

**D3. Policy-carrying placement**

传统 scheduler 只看 resource requests、taints/tolerations、affinity、quota。Agent Realm 还需要：

- network egress scope
- credential scope
- filesystem/object ACL
- secret mount policy
- human approval gates
- irreversible side-effect policy
- audit/provenance retention

所以 AKernel 的调度决策不是“哪个节点有 CPU”，而是“哪个节点/realm 能安全、快速、可审计地执行这个 agent action”。

**D4. Provenance as a scheduling input**

ExoFlow 的 nondeterminism annotation 对 agent 非常重要。AKernel 可以把 tool/action 标注成：

- deterministic/replayable
- idempotent/retriable
- external-side-effect
- requires-checkpoint-before
- requires-human-approval
- credential-bearing
- commit/abort boundary

这些 metadata 会影响调度、checkpoint、migration、retry、audit，而不是只影响日志。

## 3. 高并发下的实例冷启动与水平 Scaling

### 3.1 Agent 不应只测 cold start，而应测 useful start

传统 cold start 常见终点：

- VM booted
- container process started
- sandbox ready
- function handler invoked
- shell ready

Agent 的真实终点应是：

> `useful_start = T(first useful action completes) - T(agent execution request accepted)`

不同 agent workload 的 useful action 可以定义为：

| Workload | useful action |
|---|---|
| coding agent | `git status` 在目标 repo 中成功返回；首个测试/构建命令开始产生有效输出 |
| terminal agent | 首个 command 在正确 workspace/权限/network 下成功执行 |
| browser agent | browser profile/DevTools/页面状态 ready，首个 DOM action 成功 |
| cloud ops agent | kubeconfig/secret/network policy ready，首个只读诊断命令成功 |
| RL rollout | environment state restored，首个 step/action 成功并可记录 reward |

Agent useful start 的关键路径：

1. request accepted / realm admission
2. scheduler placement
3. sandbox/microVM/container create or restore
4. image/layer/package/browser profile hydration
5. workspace/checkpoint/object mount
6. network policy / credential / secret injection
7. PTY/tool server/browser/LSP/dev server ready
8. first useful command/tool/action completes

AKernel 的主张应是：只优化第 3 步不够，必须跨 execution plane、image plane、object/state plane、policy plane、control plane 一起优化。

### 3.2 关键相关工作

| 年份 | 来源 | 系统/机制 | URL | 核心机制/定量结果 | 与 AKernel 的关系 |
|---|---|---|---|---|---|
| 2016 | FAST | Slacker | https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter | lazy Docker image；论文报告 pull/copy 占启动 76%，实际 useful data 仅 6.4%；开发周期 20x、部署 5x 加速 | Agent image/workspace 不应整包拉取，应按 useful working set 拉取。 |
| 2020 | NSDI | Firecracker | https://www.usenix.org/conference/nsdi20/presentation/agache | microVM for serverless isolation；兼顾启动时间、密度与安全 | AKernel microVM sandbox baseline。 |
| 2020 | ASPLOS | Catalyzer | https://ipads.se.sjtu.edu.cn/_media/publications/catalyzer-asplos20.pdf | checkpoint/restore + on-demand state recovery + `sfork`；最好情况亚 ms startup | 直接对应从 warm template fork 出 agent sandbox/process。 |
| 2020 | EuroSys | SEUSS | https://www.cs.bu.edu/~jappavoo/Resources/Papers/seuss.pdf | unikernel snapshot + page sharing；报告新函数 burst throughput 51x，function deployment 从数百 ms 到 <10ms | 说明 snapshot 要与内存共享和密度一起设计。 |
| 2022 | EuroSys | FaaSnap | https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf | Firecracker snapshot + working-set prefetch + concurrent paging；端到端最多 3.5x，平均只比内存缓存 snapshot 慢 3.5% | 支持“远端/磁盘 snapshot 接近 warm memory pool”的 AKernel state plane 设计。 |
| 2023 | USENIX ATC | AWS Lambda on-demand container loading | https://www.usenix.org/conference/atc23/presentation/brooker | block-level demand loading、cache、dedup、erasure coding、convergent encryption；支持 10GiB images，单客户 15K containers/s，低至 50ms startup | 大规模 agent 扩容会打爆 registry；需要 block/layer/object 级去重和缓存。 |
| 2023 | OSDI | Mitosis | https://www.usenix.org/conference/osdi23/presentation/wei-rdma | RDMA co-designed remote fork for serverless；避免传统 provisioned concurrency | 可作为跨节点快速 fork/restore 的参考，但 agent 要额外处理 workspace/policy/provenance。 |
| 2024 | EuroSys | Pronghorn | https://www.cs.purdue.edu/homes/pfonseca/papers/eurosys24-pronghorn.pdf | checkpoint orchestration for serverless hot-start | AKernel 可借鉴 checkpoint orchestration，但需要 agent object consistency。 |
| 2024 | USENIX ATC | PASS | https://www.usenix.org/system/files/atc24-pang.pdf | PMEM-aware hypervisor + zero-copy on-demand paging；报告 SnapStart 比 Firecracker-on-PMEM 降 72%，比 FaaSnap 降 47% | 高并发 snapshot restore 的强 baseline。 |
| 2024 | OSDI | Sabre | https://www.usenix.org/conference/osdi24/presentation/lazarev | microVM snapshot 硬件加速压缩/解压 + prefetch；报告 snapshot 压缩最高 4.5x、memory restore 最高 55% | 若 AKernel 强调 snapshot 存储/网络成本，可引用压缩与预取。 |
| 2024 | ASPLOS | RainbowCake | https://github.com/IntelliSys-Lab/RainbowCake-ASPLOS24 | layer-wise container caching/sharing + sharing-aware keep-alive；OpenWhisk 启动延迟降 68%，内存浪费降 77% | Agent sandbox 多镜像共享基础层、工具层、repo 层，可做 layer-aware pool。 |
| 2025 | OSDI | AFaaS / Fork in the Road | https://www.usenix.org/conference/osdi25/presentation/chai-xiaohu | 生产级 serverless cold-start 分解：control path、contention、user init；AFaaS 使用 FRI、资源池、seeding，生产运行 18+ 月 | 强支撑 AKernel 不能只优化 restore，要处理控制面与并发 contention。 |
| 2021-2026 | containerd / Nydus / SOCI | lazy image loading | https://github.com/containerd/stargz-snapshotter, https://nydus.dev/, https://github.com/awslabs/soci-snapshotter | lazy pulling、remote snapshotter、chunk dedup、prefetch、EROFS/FUSE | AKernel 的 agent image、工具链、浏览器 profile 和 package cache 可按需加载。 |
| 2026 | Grab Engineering | SOCI/eStargz production | https://engineering.grab.com/docker-lazy-loading | Kubernetes lazy image loading 生产调优；报告 fresh node image pull 4x，P95 startup 改善 29-34%，下载 60s 到 24s | 很适合作为行业证据：lazy loading 的收益应看 P95 useful startup。 |
| 2026 | Google Cloud Docs | GKE Agent Sandbox Pod Snapshots | https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox-pod-snapshots | agent sandbox pod snapshot/restore、pre-warmed snapshot、pause/resume | 直接证明 agent sandbox snapshot 已成为云原语。 |
| 2026 | Cloudflare | Cloudflare Sandboxes GA | https://blog.cloudflare.com/sandbox-ga/ | agent coding workload：repo clone/build/dev server/credential injection/PTY/persistent processes/snapshots/Active CPU Pricing | 工业侧已经把 sandbox lifecycle、snapshot、active CPU cost productize。AKernel 需更进一步做 cluster scheduling。 |
| 2026 | E2B Docs | Sandbox metrics | https://e2b.dev/docs/sandbox/metrics | CPU/memory/disk metrics，按 sandbox compute resources 计费 | 可作为 product baseline，但指标粒度还不够到 tool/realm/object。 |
| 2026 | Modal Docs | Sandboxes / memory snapshots | https://modal.com/docs/guide/sandboxes, https://modal.com/docs/guide/memory-snapshots | untrusted code containers、memory snapshots、warm execution | 可作为 black-box sandbox baseline。 |
| 2026 | Cube Sandbox | snapshot/clone/rollback benchmark | https://cubesandbox.com/blog/posts/2026-06-03-cubesandbox-v0.3.0-snapshot.html | 面向 RL/high-concurrency workloads 的 snapshot/from-snapshot/rollback/clone | 可作为 AKernel C/R/fork baseline。 |

### 3.3 AKernel 的 horizontal scaling 设计空间

**H1. Warm pool 不应只按 image 维护，还应按 realm/object/policy 维护**

传统 warm pool 维护“预热的 runtime”。Agent warm pool 应区分：

- base runtime warm：Python/Node/Browser/shell ready
- image layer warm：language/toolchain layer cached
- object warm：repo/checkpoint/package cache/browser profile local
- policy warm：network namespace/egress/proxy/secret scope prepared
- realm warm：session state, PTY, dev server, LSP, background process preserved

**H2. Snapshot/restore 必须与 Agent Object consistency 绑定**

Agent snapshot 不只是 memory snapshot。它可能包含：

- filesystem diff
- running process tree
- open PTY/session
- browser profile
- tool server state
- package cache
- credential handle
- network/proxy state
- provenance log

AKernel 需要定义哪些对象是 restore-critical，哪些可 lazy hydrate，哪些必须 scrub，哪些必须在 commit/abort 时写回。

**H3. Burst scaling 要避免 registry/object-store/control-plane thundering herd**

高并发 agent rollout 可能同时拉取相同 repo、相同 base image、相同 package cache、相同 checkpoint template。AKernel 的扩容路径应包含：

- node-local object cache
- rack/region cache
- P2P/layer distribution
- snapshot fanout
- prefetch based on realm plan
- admission throttling based on object-store pressure
- fast path state update + async provenance writeback

**H4. Useful start 指标应分解报告**

建议 AKernel 实验报告：

| 子阶段 | metric |
|---|---|
| control path | request accepted -> placement decided |
| execution start | placement -> sandbox/microVM/container running |
| state restore | sandbox running -> checkpoint/workspace mounted |
| object hydration | bytes read, remote bytes, cache hit, page faults before first action |
| policy setup | network/secret/proxy setup latency |
| tool readiness | shell/browser/LSP/dev server/tool RPC ready |
| first useful action | first command/action success latency |

## 4. Agent 单实例的垂直 Scaling 与动态资源需求

### 4.1 为什么传统 VPA 不够

传统 vertical scaling 的目标是“根据历史 CPU/memory usage 调整 container requests/limits”。Agent 的单实例资源问题不同：

- agent 是一个有状态 process tree，不是稳定 service replica。
- 资源变化由 tool phase 驱动：edit/search/test/build/browser/package install/network IO/wait/checkpoint。
- 外部等待期间 CPU 低，但 workspace、cache、network lease、credentials、PTY 可能仍有价值。
- build/test/browser 会产生短时 CPU/memory/IO/network spike，平均值不代表 peak 风险。
- 失败、OOM、retry、rollback 会影响 task success 和 cost-to-solution。
- checkpointable/recomputable/pinned state 决定资源能否被回收，而不仅是当前 utilization。

一句话：

> VPA resizes containers; AKernel manages the resource life of an agent process.

### 4.2 关键相关工作

| 年份 | 来源 | 系统/机制 | URL | 核心数据/机制 | 局限与 AKernel 机会 |
|---|---|---|---|---|---|
| 2026 | arXiv | AgentCgroup | https://arxiv.org/abs/2602.09345 | sandboxed coding agents；144 SWE-rebench tasks；报告 OS/tool/container/init 占端到端 56-74%，memory 而非 CPU 是并发瓶颈，tool-call memory spike 最高 15.4x peak/avg；用 cgroup/eBPF/sched_ext/memcg_bpf_ops 做 tool-call-aligned control | 最接近本方向；但主要在单 sandbox/session 内，没有 realm/object/checkpoint/datacenter placement。 |
| 2026 | arXiv | Agentic AI Workload Characteristics | https://arxiv.org/abs/2605.26297 | ReAct/agent end-to-end characterization；有 turn/tool temporal structure 和长尾 | 可引用 agent phase/heavy-tail 事实；但本文要避免转向 LLM serving。 |
| 2026 | arXiv | TraceLab | https://arxiv.org/abs/2606.30560 | 约 4,300 coding-agent sessions、350K LLM steps、430K tool calls；真实 coding-agent trace | 可引用 tool-call/human gap/heavy-tail；但 OS-level resource 需要 AKernel 自采。 |
| 2026 | Kubernetes docs | VPA + in-place resize | https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/, https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/ | Kubernetes 支持调整 container CPU/memory resources；VPA 支持 InPlaceOrRecreate | 仍是 Pod/container 粒度，不能表达 tool phase、workspace cache、checkpointability。 |
| 2026 | Kubernetes docs | QoS / ResourceQuota | https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/, https://kubernetes.io/docs/concepts/policy/resource-quotas/ | QoS class 决定 node pressure 下驱逐优先级；ResourceQuota 管 namespace aggregate cap | 适合租户配额，不适合 agent active/idle/burst/rollback phase。 |
| 2025/2026 | Kubernetes docs | Dynamic Resource Allocation | https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/ | ResourceClaim / PodSchedulingContext 用于声明和协调特殊资源 | 更适合 GPU/NIC 等设备；对 CPU/mem/IO/network burst 语义不足。 |
| 2015-2026 | Linux kernel docs | cgroup v2 + PSI | https://docs.kernel.org/admin-guide/cgroup-v2.html, https://docs.kernel.org/accounting/psi.html | cgroup v2 统一 resource control；PSI 观测 CPU/memory/IO stall | 是机制不是语义。AKernel 要把 PSI/cgroup 与 tool/process/realm 绑定。 |
| 2020 | EuroSys | Autopilot | https://research.google/pubs/autopilot-workload-autoscaling-at-google-scale/ | Google-scale workload autoscaling；同时做 vertical CPU/RAM limits 与 horizontal task count，减少 limit 与 usage slack | 面向历史稳定 workload；AgentCgroup 说明 agent demand 跨 task/run/model 难预测。 |
| 2017 | SOSP | Resource Central | https://www.microsoft.com/en-us/research/publication/resource-central-understanding-predicting-workloads-improved-resource-management-large-cloud-platforms/ | Azure VM telemetry 与 workload prediction，服务 scheduler/power/migration managers | VM/workload 级预测；agent 的分支/重试/等待/工具阶段更细粒度。 |
| 2014 | ASPLOS | Quasar | https://mast.stanford.edu/pubs/quasar_resource_efficient_and_qos_aware_cluster_management/ | QoS-aware cluster management；200-server EC2 cluster 上 steady-state utilization 提升 47% | 面向 application QoS classification，不是 stateful agent process。 |
| 2024 | SOSP / arXiv | Dirigent | https://arxiv.org/abs/2404.16393 | 高 churn sandbox scheduling；也暴露 control plane 与 resource contention | 可作为 control-plane/resource contention 证据，但不是 agent phase control。 |
| 2025 | OSDI | AFaaS | https://www.usenix.org/conference/osdi25/presentation/chai-xiaohu | 生产 serverless cold start 包含 control path、resource contention、user init | 支持 agent 垂直弹性必须处理启动后持续 contention，而不只是 boot time。 |

### 4.3 Agent Process 的资源语义建议

建议 AKernel 将 `Agent Process` 定义为：

> agent-owned process tree + workspace diff + tool-phase state + resource contract + network/credential policy + checkpoint lineage + provenance handle。

资源 contract 不应只是 `cpu=4, memory=8Gi`。建议包括：

| Contract 字段 | 含义 |
|---|---|
| baseline reservation | 保持 agent runtime、shell、PTY、tool server、minimal workspace metadata 存活的资源 |
| burst envelope | build/test/browser/package install 等短时 spike 的 CPU/memory/IO/network 上限 |
| burst duration | 允许 burst 持续多久，超过后降级/排队/迁移/提示 |
| idle lease | external wait/human wait/LLM wait/API wait 时保留哪些状态、释放哪些资源 |
| checkpointability | 当前 phase 是否可 checkpoint/evict/restore/recompute |
| pinned phase | 不可中断段，例如正在写外部系统、提交 artifact、执行不可重试命令 |
| recoverable memory | 可通过 checkpoint/lazy restore 回收的内存 |
| object IO budget | workspace/checkpoint/package/browser cache 的读写预算和优先级 |
| network budget | egress/connectivity 的 scope、rate、audit level |
| OOM policy | retry from checkpoint、expand burst、fail-fast、commit partial artifact |

### 4.4 推荐的两级控制循环

| 控制层 | 时间尺度 | 信号 | 动作 |
|---|---|---|---|
| in-kernel / node fast path | ms-subsecond | cgroup usage, PSI, tool-call boundary, process spawn, major faults | adjust cgroup weights/limits, IO priority, kill/retry child process, mark pressure |
| sandbox daemon | subsecond-seconds | phase hints, process tree, workspace dirty bytes, checkpointability | checkpoint, hibernate, restore, grow/shrink local lease, throttle tool |
| realm scheduler | seconds-minutes | realm SLO, object locality, quota, cost, queue pressure | migrate realm, place next process near object, rebalance warm pools, admit/throttle burst |
| control plane | minutes+ | audit/provenance, tenant quota, policy changes, historical profiles | update policies, model future warm templates, summarize trace |

这样可以把 AgentCgroup 的 fast path 与 AKernel 的 datacenter scheduler 连接起来。

### 4.5 与传统 autoscaling 的论文对比

| 维度 | VPA / Autopilot / Resource Central | AKernel phase-aware elasticity |
|---|---|---|
| 对象 | VM / container / Pod / replica | Agent Process inside Agent Realm |
| 信号 | 历史 CPU/mem utilization、percentile、QoS | tool phase、process tree、PSI、workspace dirty state、checkpointability、policy |
| 时间尺度 | 秒到分钟，常需要重建或 in-place resize | ms fast path + seconds restore/hibernate + realm-level scheduling |
| 动作 | 调整 request/limit、replica、placement | burst lease、idle lease、checkpoint、hibernate、restore、migrate、tool throttling |
| 正确性 | 满足服务 SLO，不被 OOM/evict | 保持 workspace、credentials、side effects、artifact provenance、retry semantics |
| 目标 | utilization / SLO / cost | agent-goodput / cost-to-solution / time-to-useful-action / retry loss |

可直接写进论文的 contrast：

> Traditional vertical autoscaling resizes containers based on resource history. AKernel performs phase-aware vertical elasticity for agent processes: it changes resource leases at tool-call and lifecycle boundaries, while preserving workspace state, checkpoint lineage, policy constraints, and provenance.

## 5. 统一视角：AKernel 的三层抽象如何覆盖缺口

### 5.1 Agent Realm

Realm 解决 distributed scheduling 的单位问题：

- admission and quota
- group placement
- multi-agent/shared workspace
- policy boundary
- audit/provenance boundary
- warm pool / checkpoint lineage
- cost and SLO accounting

它应该成为 AKernel scheduler 的核心对象，而不是只调 Pod 或 sandbox。

### 5.2 Agent Object

Object 解决 useful start 和 locality 问题：

- repo
- checkpoint
- filesystem diff
- package cache
- browser profile
- dataset
- build/test artifacts
- logs/traces
- credential-bearing handles

Object metadata 决定是否 prefetch、lazy hydrate、cache、replicate、scrub、commit、rollback。

### 5.3 Agent Process

Process 解决 vertical elasticity 问题：

- process tree
- cgroup hierarchy
- tool-call boundary
- resource lease
- checkpointability
- network/secret policy
- provenance handle

它不是普通 container。它是 stateful, policy-carrying, artifact-producing execution unit。

## 6. 建议 AKernel 论文中的系统设计

### 6.1 Control Plane

职责：

- Realm admission
- policy validation
- quota/fairness
- SLO/cost budget
- provenance persistence
- async reconciliation with Kubernetes

关键点：不要把高频 tool/process state 都同步进 etcd/API Server。KubeDirect/Dirigent 已经证明高 churn path 会成为瓶颈。

### 6.2 Execution Plane

职责：

- sandbox create/restore/fork/hibernate
- process tree management
- cgroup/eBPF/PSI control
- PTY/tool server/browser lifecycle
- local checkpoint

关键点：把 AgentCgroup 的 tool-call resource control 变成 node-local primitive。

### 6.3 Object/State Plane

职责：

- Agent Object namespace
- local/rack/region cache
- checkpoint lineage
- COW workspace
- lazy hydration/prefetch
- commit/rollback
- object provenance

关键点：useful start 的关键不是“sandbox ready”，而是“关键对象 ready”。

### 6.4 Policy/Network Plane

职责：

- egress policy
- credential/secret scope
- proxy/session routing
- irreversible side-effect guard
- audit log
- policy-carrying migration

关键点：policy 是调度和恢复条件，不只是运行时拦截。

### 6.5 Deployment Plane

职责：

- Build Your Own Cluster
- Terraform/Helm
- multi-cloud sandbox pools
- preheated image/object/snapshot distribution
- node capability inventory

关键点：agent 应该能“像调用函数一样驱动数据中心”，而不是只调用一个 sandbox API。

## 7. Evaluation 建议

### 7.1 Workloads

| Workload | 来源 | 目的 |
|---|---|---|
| Coding agent | SWE-bench Verified / SWE-rebench / internal coding-agent trace | repo/object locality、build/test burst、checkpoint/retry |
| Terminal agent | Terminal-Bench / internal terminal tasks | shell/process tree/file/network control |
| Browser/computer-use | OSWorld / BrowserGym / WebArena / WorkArena | browser profile、GUI state、network policy、rollback |
| Cloud ops agent | AIOpsLab / Cloud-OpsBench / internal K8s ops | credential/egress/audit/provenance |
| RL rollout/evaluation | SWE-Gym / Terminal-Bench Harbor style rollout | high-concurrency fork/restore/useful start |
| Multi-agent workflow | MultiAgentBench / TheAgentCompany style tasks | shared realm/object/policy/group scheduling |

注意：如果 workload 使用 LLM，也只把 LLM call 当外部 service，评估重点放在 sandbox/resource/object/policy 层。

### 7.2 Baselines

Distributed scheduling baselines：

- Kubernetes Pod/Job
- Kueue for batch admission
- Kubernetes Agent Sandbox CRD if available
- Knative/OpenFaaS-style FaaS baseline
- Ray/KubeRay actor baseline
- openYuanRong baseline
- AKernel without realm-aware scheduling
- AKernel without object locality
- AKernel without policy-aware placement

Horizontal scaling baselines：

- cold container pull/start
- pre-pulled image
- stargz/SOCI/Nydus lazy image
- Firecracker cold boot
- Firecracker snapshot restore
- warm pool
- snapshot pool
- fork/clone pool
- P2P/preheated registry
- E2B/Daytona/Modal/GKE/Cloudflare/Cube as black-box product reference when feasible

Vertical scaling baselines：

- static container request/limit
- Kubernetes VPA/in-place resize
- cgroup-only policy
- AgentCgroup-like tool-call control
- AKernel without idle lease
- AKernel without checkpoint-aware eviction
- AKernel without phase hints

### 7.3 Metrics

**Core metrics**

- task goodput: successful verified tasks / resource-hour
- cost-to-solution
- time-to-useful-action
- p50/p95/p99 useful start
- resource-hours saved
- OOM/retry rate
- checkpoint lost work
- policy violation/block rate
- provenance completeness

**Distributed scheduling metrics**

- placement latency
- realm admission throughput
- sandboxes/s cluster-wide
- object cache hit rate
- cross-node/rack/region traffic
- checkpoint locality hit rate
- policy-compatible placement success rate
- control-plane writes per action

**Horizontal scaling metrics**

- create-to-shell
- create-to-first-command
- create-to-repo-ready
- create-to-browser-ready
- first useful action latency
- registry/object-store bytes
- snapshot bytes read before first action
- page faults before first action
- burst fanout completion time for N sandboxes
- idle warm pool memory cost

**Vertical scaling metrics**

- cgroup update latency
- PSI stall time
- tool p95/p99 latency
- peak/avg memory ratio by tool
- memory pressure induced retries
- idle lease reclaimed memory
- checkpoint/hibernate latency
- restored useful-action latency
- resource attribution by realm/tool/object

### 7.4 Experiments

**E1. Realm-aware placement**

Run mixed coding/browser/cloud-ops workloads with shared repos/checkpoints/package caches. Compare Kubernetes/openYuanRong baseline vs AKernel scheduler. Measure object cache hit, cross-node traffic, p99 task latency, cost-to-solution.

**E2. Useful-start scaling**

Burst from 1 to 10K agent sandboxes under four modes: cold image, lazy image, snapshot restore, AKernel object-aware useful start. Measure p50/p95/p99 useful start and bytes read before first action.

**E3. Snapshot/locality fanout**

Fork many realms from a shared template checkpoint. Compare local snapshot, remote snapshot, P2P distribution, object-store restore. Measure fanout throughput, network egress, restore tail.

**E4. Phase-aware vertical elasticity**

Run coding/terminal/browser tasks with static quota, VPA/in-place resize, AgentCgroup-like tool control, and AKernel resource contracts. Measure OOM/retry, resource-hours, tool stall, success rate.

**E5. Idle/wait hibernation**

Inject external wait/human wait/API wait gaps. Compare keeping warm sandboxes vs hibernate/restore with object-locality. Measure reclaimed memory, restore-to-useful-action, lost work.

**E6. Policy-aware scheduling**

Use workloads with credential-bearing actions, restricted egress, external side effects. Compare policy as runtime check only vs policy-carrying placement. Measure rejected placements, migration failures, audit completeness, overhead.

## 8. 可直接放进论文的 Related Work 组织

### 8.1 Cloud OS and serverless distributed applications

Use SigmaOS and YuanRong to motivate cloud OS / distributed serverless substrate. AKernel differs by making agent realm/object/process/policy first-class.

### 8.2 Serverless orchestration and workflow scheduling

Use Dirigent, KubeDirect, ORION, ExoFlow, AlloyStack, KubeAdaptor. The contrast:

> These systems optimize functions, workflows, or Kubernetes control paths. AKernel optimizes stateful agent execution domains whose placement depends on objects, checkpoints, policies, and provenance.

### 8.3 Sandbox cold start and snapshotting

Use Firecracker, Catalyzer, SEUSS, FaaSnap, Lambda on-demand container loading, PASS, Sabre, RainbowCake, AFaaS, lazy image systems. The contrast:

> These works reduce boot, restore, or image loading latency. AKernel uses them as mechanisms, but optimizes time-to-useful-action for agent workloads.

### 8.4 Resource control and vertical autoscaling

Use AgentCgroup, cgroup v2/PSI, Kubernetes VPA/in-place resize, Autopilot, Resource Central, Quasar. The contrast:

> AgentCgroup identifies tool-call-level resource dynamics in individual agent sandboxes. AKernel extends this insight to datacenter-scale realm scheduling, object-aware resource attribution, and checkpoint-aware vertical elasticity.

### 8.5 Industry agent sandbox platforms

Use GKE Agent Sandbox, Cloudflare Sandboxes, E2B, Modal, Cube, Docker Sandboxes, OpenAI/Anthropic docs. The contrast:

> Industry systems show that sandbox APIs, snapshots, metrics, and active CPU pricing are becoming commodity. AKernel addresses the missing system layer above sandbox creation: scheduling, useful-start scaling, resource elasticity, object locality, and provenance across a cluster.

## 9. 优先引用清单

如果篇幅有限，建议优先引用：

1. AgentCgroup: https://arxiv.org/abs/2602.09345  
   核心 OS/resource 证据。
2. Dirigent: https://arxiv.org/abs/2404.16393  
   高 churn sandbox control-plane/scheduling 证据。
3. KubeDirect: https://www.usenix.org/system/files/nsdi26-qi.pdf  
   Kubernetes 兼容与 fast path 的张力。
4. AFaaS / Fork in the Road: https://www.usenix.org/conference/osdi25/presentation/chai-xiaohu  
   生产 cold start 分解和 hidden bottleneck。
5. AWS Lambda on-demand container loading: https://www.usenix.org/conference/atc23/presentation/brooker  
   大规模 image/container loading 证据。
6. FaaSnap: https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf  
   snapshot restore / working set prefetch。
7. Slacker: https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter  
   lazy image/useful working set 的经典证据。
8. ExoFlow: https://www.usenix.org/conference/osdi23/presentation/zhuang  
   nondeterminism/provenance/recovery 相关。
9. AlloyStack: https://madsys.cs.tsinghua.edu.cn/publication/alloystack-a-library-operating-system-for-serverless-workflow-applications/eurosys25-you.pdf  
   workflow locality/LibOS/引用传递。
10. Kubernetes Agent Sandbox: https://agent-sandbox.sigs.k8s.io/docs/  
    工业/生态正在 agent sandbox API 化。
11. GKE Agent Sandbox Pod Snapshots: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/agent-sandbox-pod-snapshots  
    agent sandbox snapshot 产品化。
12. Cloudflare Sandboxes GA: https://blog.cloudflare.com/sandbox-ga/  
    agent coding sandbox productization and active CPU pricing。
13. Kubernetes VPA / in-place resize: https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/  
    传统 vertical scaling baseline。
14. Linux cgroup v2 / PSI: https://docs.kernel.org/admin-guide/cgroup-v2.html, https://docs.kernel.org/accounting/psi.html  
    AKernel fast resource control substrate。

## 10. 建议写进 Paper_outline.md 的新段落

### Motivation paragraph

AI agent infrastructure is quickly converging on sandbox APIs: developers can now create isolated containers or microVMs, execute commands, preserve files, and restore snapshots. However, sandbox creation is no longer the hard systems question. A datacenter-scale agent workload consists of thousands of stateful, bursty, policy-carrying execution domains. Their performance depends on where workspaces, checkpoints, caches, credentials, and audit logs are placed, when sandboxes can be forked or hibernated, and how resources change across tool-call phases. Existing cloud schedulers manage pods, jobs, or functions; existing autoscalers resize containers; existing sandbox services expose lifecycle APIs. None of them provide a cloud OS abstraction for scheduling and elastically managing agent execution.

### Contribution paragraph

AKernel introduces three cloud OS abstractions for datacenter-use agents. An **Agent Realm** is the scheduling and isolation domain for an agent task or workflow. An **Agent Object** represents repositories, checkpoints, logs, package caches, browser state, datasets, and other execution artifacts with locality and policy metadata. An **Agent Process** is a stateful, policy-carrying process tree with phase-aware resource contracts. Together, these abstractions let AKernel optimize time-to-useful-action, task goodput, object locality, resource-hours, and provenance completeness.

### Non-goal paragraph

AKernel does not optimize LLM inference or model serving. It treats model calls as external services and focuses on the operating-system substrate around agents: sandbox lifecycle, distributed placement, state and object management, resource elasticity, policy enforcement, and auditability.

## 11. 本轮 Survey 的最终结论

AKernel 最应该占住的论文定位不是“更快的 sandbox”，也不是“agent serving scheduler”，而是：

> A cloud-native operating system for agent execution that turns sandboxed agents into schedulable, stateful, policy-carrying datacenter processes.

三条技术贡献可以直接对应用户提出的问题：

1. **云原生 agent 的分布式调度**  
   提出 Realm-aware scheduler，把 placement 从 pod/function/job 升级为 realm/object/policy/provenance-aware scheduling。

2. **高并发实例冷启动与水平 scaling**  
   提出 useful-start pipeline，把 cold start 从 boot/restore 指标升级为 first useful agent action，并通过 snapshot、lazy object hydration、warm realm/object pools、P2P/cache fanout 优化。

3. **Agent 单实例的垂直 scaling**  
   提出 phase-aware vertical elasticity，把 AgentCgroup 的 tool-call resource insight 扩展为 Agent Process resource contract，支持 baseline/burst/idle/pinned/checkpointable/recoverable resource states。

这三条合在一起，能把 AKernel 从“sandbox 平台”提升成 OSDI/SOSP 论文需要的系统抽象与端到端 argument。
