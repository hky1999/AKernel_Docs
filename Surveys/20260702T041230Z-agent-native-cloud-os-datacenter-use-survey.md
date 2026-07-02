# AKernel Agent-Native Cloud OS Survey

调研完成时间：2026-07-02 04:12:30 UTC

调研主题：面向 OSDI/SOSP 的 AKernel 论文定位、相关工作、创新点风险与证据需求。

本次调研依据：

- 本地论文材料：`AGENTS.md`, `AKernel.md`, `Paper_outline.md`, `AKernel_Paper_Structure.md`, `outline_draft.md`
- 本地参考论文：
  - `References/Szekely et al_2024_Unifying serverless and microservice workloads with SigmaOS.pdf`
  - `References/Chen 等 - 2024 - YuanRong A Production General-purpose Serverless System for Distributed Applications in the Cloud.pdf`
  - `References/Chai et al_2025_Fork in the Road.pdf`
  - `References/Zheng 等 - 2026 - AgentCgroup Understanding and Controlling OS Resources of AI Agents.pdf`
- 并行 subagent 调研：
  - 系统方向：cloud OS、cluster scheduling、serverless、cold start、stateful serverless、image distribution
  - AI sandbox 方向：E2B、Modal、Daytona、OpenSandbox、Cube、GKE/Azure/Cloudflare sandbox、AgentCgroup/Crab/AgentTrust
  - Agent workload 方向：AgentBench、WebArena、WorkArena、OSWorld、SWE-bench、Terminal-Bench、multi-agent、AIOS、TraceLab、Agentic AI Workload Characteristics

## 0. 直接结论

你的判断基本是对的：AgentCgroup 这类 2026 年工作已经开始把 AI agent 放进 OS 资源管理视角里，但它主要研究的是 sandboxed coding agent 在一批 benchmark task 上的 OS-level resource dynamics 和 tool-call-aligned resource control。它的 workload 单元仍然是一个 agent/session/sandbox 内部的 resource timeline，最多扩展到多租户隔离；它没有把大量 agent node、agent realm、agent object locality、policy/provenance、checkpoint/restore、跨节点 placement 和 BYOC cluster 统一成 agent-native cloud OS。

但是，仅仅说“我们有一大堆 agent node，会组成 agent OS”还不够达到 OSDI/SOSP 的创新要求。OSDI/SOSP 会要求你证明：

1. 这是一个真实、普遍、增长中的系统问题，不只是一个工程愿景。
2. 现有 cloud OS/serverless/sandbox/agent framework 为什么不能自然解决它。
3. AKernel 提出的抽象和机制不是简单组合，而是由 agent workload 的新性质驱动。
4. 系统实现跨越 control plane、execution plane、data plane、state plane、policy plane 后，能在端到端指标上显著改善。
5. evaluation 不能只比 cold start，要证明 agent-goodput、cost-to-solution、time-to-useful-action、state/object locality、resource-hours saved、policy/provenance overhead。

更准确的论文定位应该是：

> Existing systems give each agent a sandbox, a tool runtime, or a serverless function substrate. AKernel treats large numbers of stateful, artifact-producing, policy-carrying agents as cloud processes, and provides an agent-native cloud OS that co-designs lifecycle, placement, object locality, checkpoint/restore, policy, provenance, and cluster deployment.

这条线符合 OSDI/SOSP 的潜在创新点要求，但必须用 trace 和实验把它钉住。否则会被审稿人打成“把 serverless、Kubernetes、sandbox、object store、checkpoint、eBPF、Helm 拼在一起的系统集成”。

## 1. 对现有论文大纲的分析

### 1.1 当前大纲最强的部分

当前大纲已经避开了一个危险误区：没有把 AKernel 写成 “E2B-like sandbox SDK”。这是正确的。E2B、Modal、Daytona、OpenSandbox、Cube Sandbox、Cloudflare Sandboxes、Azure Container Apps Sandboxes、GKE Agent Sandbox 都已经公开提供了相当多 sandbox primitive，例如 create/exec/files/ports/network policy/snapshot/fork/pause/resume/BYOC/Kubernetes integration。

因此，AKernel 的论文贡献不能是：

- 我们可以启动一个 sandbox。
- 我们可以给 agent 一个 Linux/desktop/code interpreter。
- 我们支持 exec/file/network/snapshot。
- 我们启动更快。

当前大纲把主张放到 Agent Process、Agent Realm、Agent Object、Datacenter Function，以及 state/data/policy/deployment 的统一语义上，这是正确方向。

### 1.2 当前大纲仍然最缺的证据

大纲里最关键但还没有完成的部分是 workload characterization。没有它，论文主张会显得像系统愿景，而不是系统发现。

需要的最小 trace 不是“跑几个 benchmark 看资源变化”，而是 realm/session/process/object/tool/policy 多层 trace：

| Trace 层级 | 必须采集的字段 | 支撑的论文论点 |
|---|---|---|
| Agent Realm | session duration, status, success metric, retry count, human intervention, cost, SLO | Agent task 是有生命周期和成功率的整体，不是单次 invocation |
| Agent Process | create/start/fork/checkpoint/restore/hibernate/migrate/commit/fail, runtime, node, image, resources | agent execution 需要 OS lifecycle |
| Agent Object | object type, size, producer/consumer, reuse distance, cache hit, node location, read/write count | agent state/artifact 是一等对象，调度要跟着状态走 |
| Tool/Event | tool call type, shell/file/network/LLM/API duration, input/output size, side effects | tool-call boundary 是资源、策略、checkpoint 的关键边界 |
| Policy/Provenance | credential use, egress domain, allow/block, artifact refs, command refs | agent 执行需要 capability 和 audit 贯穿执行生命周期 |

这些 trace 应该形成 5 个 workload insight：

1. Agent workloads are stateful and long-lived.
2. Agent workloads are bursty and branchy.
3. Agent workloads are artifact-heavy.
4. Agent workloads are idle-prone and wait on LLM/tools/humans/external APIs.
5. Agent workloads are policy-constrained and provenance-sensitive.

### 1.3 大纲中应该强化的主线

建议把论文主线压缩为三层闭环：

1. Characterization：真实 agent workload 暴露新系统问题。
2. Abstraction：Agent Process / Realm / Object / Datacenter Function 将这些问题提升为 OS 语义。
3. Cross-layer implementation：AKernel 将调度、sandbox、object store、C/R、image loading、policy/provenance、BYOC deployment 连成一个 agent-native cloud OS。

论文贡献建议写成 3 条，而不是 4 条：

1. Production characterization of datacenter agent workloads.
2. Agent-native cloud OS abstractions and lifecycle semantics.
3. AKernel implementation and evaluation showing end-to-end agent goodput/cost/locality benefits.

## 2. 相关工作分类总览

下面按类别和完成/发表时间排序。表中“AKernel 关系”重点写该工作能支持什么论点，以及 AKernel 必须如何区分自己。

## 3. Cloud OS / Cluster Scheduling / Distributed Execution

| 时间 | 会议/来源 | 工作 | 核心贡献 | AKernel 关系 |
|---|---|---|---|---|
| 2013 | EuroSys | Omega: Flexible, Scalable Schedulers for Large Compute Clusters | 共享集群状态，多调度器乐观并发提交，解决大集群调度扩展性。 | AKernel 可借鉴共享状态和多调度器，但 agent placement 还要感知 object/checkpoint/image/policy。https://research.google/pubs/omega-flexible-scalable-schedulers-for-large-compute-clusters/ |
| 2015 | EuroSys | Borg: Large-scale Cluster Management at Google | Google 生产集群管理，资源打包、优先级、超售、故障恢复。 | Borg 是 cluster OS 先例；AKernel 可定位为 agent-native cloud OS，而非通用 batch/service manager。https://research.google.com/pubs/pub43438.html |
| 2018 | OSDI | LegoOS | 分布式 splitkernel，将处理器、内存、存储解耦管理。 | 支撑“OS 可以扩展到数据中心级资源管理”的叙事；AKernel 资源对象是 agent/process/object/policy。https://www.usenix.org/system/files/osdi18-shan.pdf |
| 2018 | OSDI | Ray | task + actor 统一支撑 AI/RL 分布式应用，包含对象存储和容错。 | Ray 是 AI application runtime，不是 OS；AKernel 可支持 Ray-like/agentic workloads 的隔离、状态和权限。https://www.usenix.org/system/files/osdi18-moritz.pdf |
| 2020 | EuroSys | Borg: The Next Generation | 公开 2019 Borg trace，展示 alloc sets、自动垂直扩缩、依赖和 workload 演化。 | AKernel 的 characterization 应学习 Borg trace 写法：先用真实 trace 证明 workload 特征。https://research.google/pubs/borg-the-next-generation/ |
| 2020 | EuroSys | Autopilot | Google 生产自动扩缩，基于历史数据降低 slack/OOM。 | agent resource demand 更受 tool/model/external API 影响，不能只用传统 workload history。https://research.google/pubs/autopilot-workload-autoscaling-at-google-scale/ |
| 2023 | NSDI | SkyPilot | 跨云 broker，为 ML/batch jobs 自动选择云、区域和实例，优化成本/等待时间。 | 支撑 BYOC/multi-cloud 是系统问题；AKernel 更进一步管理 agent lifecycle 和 state locality。https://www.usenix.org/system/files/nsdi23-yang-zongheng.pdf |
| 2024 | SOSP | SigmaOS | 用 proc/realm/object 风格的 cloud OS API 统一 serverless 和 microservice；报告低启动延迟和高 proc throughput。 | AKernel 最重要近邻。差异必须是 agent-native lifecycle：tool capability、artifact、checkpoint、provenance、multi-agent realm。https://pdos.csail.mit.edu/papers/sigmaos%3Asosp24.pdf |
| 2024 | SIGCOMM | YuanRong | 华为生产通用 serverless system，含 distributed computing kernel、data system、多语言 runtime、层次调度。 | AKernel 若基于 openYuanRong，不能把 YuanRong 原有能力当原创；贡献应是 agent process/object/state/policy 改造。https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final24-acmpaginated.pdf |
| 2025 | USENIX ATC | DEEPSERVE | Serverless LLM serving at scale，面向 NPU-centric serving 和 request-job-task 抽象。 | 说明 AI workload 正进入 serverless/cloud OS 管理面；但它面向 model serving，不是 agent execution OS。https://www.usenix.org/conference/atc25/presentation/hu-junhao |

关键结论：SigmaOS/YuanRong 已经覆盖了“cloud OS / serverless kernel”这个大叙事。AKernel 必须回答：为什么 agent workload 不是 SigmaOS/YuanRong 直接能吞掉的一类普通 workload？答案应落在 agent state、tool side effects、artifact/object lifecycle、policy/provenance、multi-agent realm、LLM/human/API waits。

## 4. Serverless Runtime / Sandbox / Cold Start / Lifecycle

| 时间 | 会议/来源 | 工作 | 核心贡献 | AKernel 关系 |
|---|---|---|---|---|
| 2018 | USENIX ATC | SAND | application-level sandboxing 和层次 message bus，优化函数链通信。 | 支撑 agent pipeline/toolchain 不能只靠独立容器。https://www.usenix.org/system/files/conference/atc18/atc18-akkus.pdf |
| 2018 | USENIX ATC | SOCK | 分析 Linux container primitive 和 Python import 开销，serverless-optimized containers 比 Docker 快。 | 说明 cold start 由 runtime/import/storage/network 等多因素构成；agent 环境依赖更重。https://www.usenix.org/conference/atc18/presentation/oakes |
| 2020 | NSDI | Firecracker | AWS microVM，为 Lambda/Fargate 提供低开销强隔离。 | AKernel sandbox baseline；但 Firecracker 不解决 agent object/policy/realm。https://www.usenix.org/conference/nsdi20/presentation/agache |
| 2020 | USENIX ATC | Faasm | WebAssembly faaslet，支持轻量隔离和状态共享。 | 可作为轻量隔离方向参考；agent 有更强 Linux/tool compatibility 要求。https://www.usenix.org/conference/atc20/presentation/shillaker |
| 2020 | EuroSys | SEUSS | unikernel snapshot stack，跳过 boot/runtime/import 重复路径。 | 支撑 snapshot/fork 作为快速 agent clone 技术；但 AKernel 要处理 side effects 和 policy cleanup。https://arxiv.org/abs/1910.01558 |
| 2020 | ASPLOS | Catalyzer | initialization-less booting 和 sandbox fork，实现 sub-ms serverless startup。 | 可支撑 AKernel agent process fork/restore 的节点侧机制。https://ipads.se.sjtu.edu.cn/pub/projects/catalyzer |
| 2022 | EuroSys | FaaSnap | Firecracker VM snapshot，优化恢复时的 page loading 和 overlap restore。 | 适合 AKernel microVM sandbox restore baseline。https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf |
| 2022 | EuroSys | Medes | serverless sandbox memory dedup，缓解 warm/cold latency 与内存成本矛盾。 | 大量同构 agent worker 可复用 runtime memory；但 agent private state/security wiping 更复杂。https://jitaogithub.github.io/publication/medes/medes_paper.pdf |
| 2024 | SOSP | Dirigent | clean-slate serverless orchestrator，移除 invocation critical path 上的持久状态更新。 | AKernel control plane 应学习它如何证明 orchestrator 是瓶颈；但 agent dependency/state/policy 更复杂。https://anakli.inf.ethz.ch/papers/dirigent_sosp24.pdf |
| 2024 | EuroSys | Pronghorn | 自动选择 checkpoint 时机和 hot-start snapshot，关注 C/R orchestration。 | AKernel checkpoint-on-idle/fork/migrate 策略可对照 Pronghorn。https://www.cs.purdue.edu/homes/pfonseca/papers/eurosys24-pronghorn.pdf |
| 2024 | USENIX ATC | Function REWIND | 请求后将容器 rewind 到初始状态，兼顾安全和 warm start。 | agent 需要区分必须清除的副作用与必须保留的记忆/产物。https://www.usenix.org/conference/atc24/presentation/song |
| 2025 | OSDI | Fork in the Road / AFaaS | Ant FaaS 生产冷启动经验，指出 control path、resource contention、user-code initialization 是端到端瓶颈。 | AKernel 应直接采用该教训：论文不能只报 VM/container boot，要报 time-to-useful-action。https://www.usenix.org/system/files/osdi25-chai-xiaohu.pdf |
| 2025 | USENIX ATC | Burst Computing | group invocation 支撑 sudden massively parallel serverless jobs。 | AKernel 的 multi-agent swarm/RL rollout 可借鉴 group-level startup 和 isolation。https://www.usenix.org/conference/atc25/presentation/barcelona-pons |

关键结论：冷启动方向已经很拥挤。AKernel 不能说“我们用 snapshot/lazy image/P2P 所以更快”作为主创新。正确写法是：agent task 的 useful start 是 `schedule + runtime + image working set + dependency import + object restore + policy setup + first tool action`，AKernel 优化这个端到端路径。

## 5. Stateful Serverless / Workflow / Data Passing

| 时间 | 会议/来源 | 工作 | 核心贡献 | AKernel 关系 |
|---|---|---|---|---|
| 2018 | OSDI | Pocket | 面向 serverless analytics 的弹性临时存储，按性能/成本自动扩缩。 | Agent Object 中的临时 artifact/cache/log/checkpoint 可借鉴 Pocket。https://www.usenix.org/system/files/osdi18-klimovic.pdf |
| 2020 | PVLDB | Cloudburst | Anna KV + executor-local cache + overlay routing，提供 stateful functions。 | Agent state locality 可对照 Cloudburst，但 AKernel state 包含文件、checkpoint、trace、policy refs。https://www.vldb.org/pvldb/vol13/p2438-sreekanti.pdf |
| 2020 | OSDI | Beldi | fault-tolerant and transactional stateful serverless workflows。 | agent 外部 action 常不可回滚，AKernel 需要 audit/compensation，而不只是 exactly-once。https://www.usenix.org/conference/osdi20/presentation/zhang-haoran |
| 2021 | USENIX ATC | SONIC | 对 serverless DAG 每条边动态选择数据传递方式，并做通信感知 placement。 | AKernel object-aware scheduling 的直接相关工作。https://www.usenix.org/conference/atc21/presentation/mahgoub |
| 2022 | ASPLOS | FaaSFlow | worker-side workflow scheduling 和 FaaStore，减少 master-side 调度和数据移动。 | agent workflow 可下沉到 node/realm manager，减少中心调度开销。https://dl.acm.org/doi/10.1145/3503222.3507717 |
| 2023 | SOSP | Halfmoon | log-optimal fault-tolerant stateful serverless，降低 exactly-once logging overhead。 | AKernel provenance/replay 需要说明哪些状态可重放，哪些外部副作用只能审计。https://sosp2023.mpi-sws.org/accepted.html |

关键结论：stateful serverless 已经证明“函数不是无状态”这个论点。因此 AKernel 不能停留在“agent 有状态”。更强的说法应是：agent state 是 multi-layer、artifact-heavy、policy-carrying、provenance-sensitive 的 OS state。

## 6. Image Lazy Loading / P2P Distribution / Provisioning

| 时间 | 会议/来源 | 工作 | 核心贡献 | AKernel 关系 |
|---|---|---|---|---|
| 2016 | FAST | Slacker | Lazy Docker containers，启动只需要拉少量镜像数据。 | AKernel distill-fs/lazy image 可用此作为经典依据。https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter |
| 2018+ | Uber OSS | Kraken | P2P-powered Docker registry，用于大规模镜像分发。 | AKernel 大规模 agent image rollout 的工程 baseline。https://github.com/uber/kraken |
| 2020 | USENIX ATC | DADI | block-level image service，按需传输并支持 P2P。 | 与 agent runtime/container/microVM image lazy loading 直接相关。https://www.usenix.org/conference/atc20/presentation/li-huiba |
| 2021 | USENIX ATC | FaaSNet | Alibaba Function Compute 上快速 provision custom serverless runtimes。 | agent runtime 更大、更异构，需结合模型/工具/package cache。https://www.usenix.org/conference/atc21/presentation/wang-ao |
| 2025 | USENIX ATC | Poby | SmartNIC-accelerated image provisioning。 | AKernel 可在 future work 提 hardware offload，但不应作为主线。https://www.usenix.org/conference/atc25/presentation/chang |
| 2026 | CNCF/工程 | Dragonfly | P2P image/file distribution。 | AKernel 可把它作为 image plane 组件，但创新点应是 locality-aware agent placement。https://d7y.io/ |

关键结论：镜像懒加载/P2P 是必要组件，不是论文主贡献。AKernel 的贡献应是把 image locality、object locality、checkpoint locality 放入同一个 agent scheduler。

## 7. AI Sandbox Products / Open Infrastructure

| 时间 | 系统 | 核心能力 | 单实例还是集群 | AKernel 关系 |
|---|---|---|---|---|
| 2023-2026 | E2B | AI agent sandbox，SDK create/connect/kill，command/files/templates 等。 | 公开抽象以单 sandbox 为主。 | 典型 commodity sandbox baseline；AKernel 不应与其比“谁有 exec/files”。https://e2b.dev/docs |
| 2025-2026 | Modal Sandboxes | secure container，exec、volumes、ports/tunnels、GPU、secrets、snapshot/fan-out。 | 单 sandbox primitive，平台后台可调度。 | AKernel 可对比其缺少公开 agent realm/object locality/provenance 语义。https://modal.com/docs/guide/sandboxes |
| 2026 | Daytona | full computer/dev environment，snapshot/fork/GPU/firewall/custom runners。 | 单 computer/sandbox 为核心，支持 runner pool。 | 比 E2B 更接近 stateful agent computer，但不是 agent OS。https://www.daytona.io/docs/en/sandboxes/ |
| 2026 | Morph Cloud | instance snapshot/branch/restore，用于 time travel/debug/parallel exploration。 | 偏 VM state branching。 | 支撑 branch/fork 对 agent 有价值；AKernel 要做 cluster/realm/object/policy。https://cloud.morph.so/docs/developers |
| 2026 | Cloudflare Sandboxes | Agent 可运行 shell/Python/Node/package install/files/background process/preview；有 egress policy、credential injection。 | 单 sandbox + Workers/Durable Objects 管理。 | policy 能力强，但不是 datacenter agent OS。https://developers.cloudflare.com/agents/tools/sandbox/ |
| 2026 | Azure Container Apps Sandboxes | on-demand microVM，sub-second startup，memory+disk snapshot，suspend/resume，SandboxGroup，VNet/egress/managed identity。 | SandboxGroup 已接近集群级 sandbox pool。 | 强相关风险。AKernel 必须强调 agent-native OS objects/locality/provenance，而不是 ARM resource group。https://sandboxes.azure.com/docs/sandboxes/ |
| 2025-2026 | GKE/Kubernetes Agent Sandbox | Kubernetes primitive/CRD，gVisor/Kata，warm pool，Pod snapshots，高速分配。 | 明确是 Kubernetes 上的 stateful singleton agent workload。 | 目前最危险相关工作之一；AKernel 要从 K8s primitive 提升到 agent OS semantics。https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke |
| 2026 | OpenSandbox | open-source AI sandbox platform，多语言 SDK、Docker/Kubernetes runtime、credential vault、egress sidecar。 | 单 sandbox API + K8s backend。 | 可作为 open-source baseline；AKernel 应避免重复“统一 sandbox API”。https://github.com/opensandbox-group/OpenSandbox |
| 2026 | Tencent Cube Sandbox | RustVMM/KVM microVM，snapshot cloning，sub-60ms cold start，多节点集群，E2B compatible，eBPF/L7 egress policy。 | 支持多节点 cluster，但以 sandbox microVM 为单位。 | 强 baseline；AKernel 差异是 agent realm/object/locality/provenance/goodput。https://cubesandbox.com/ |
| 2026 | Rivet agentOS | 轻量 agent runtime/isolate，支持 per-agent CPU/memory/network limits 和可选 sandbox。 | 多 agent runtime，但公开资料偏 runtime。 | 名称相近，需区分 AKernel 是 cloud OS，不是轻量本地/平台 runtime。https://github.com/rivet-dev/agentos |
| 2026 | OpenAI Code Interpreter | Responses API Python tool，sandboxed VM/container workspace，文件上传/生成，memory tier。 | 单 container/session primitive。 | hosted code interpreter baseline。https://developers.openai.com/api/docs/guides/tools-code-interpreter |
| 2026 | OpenAI Agents SDK Sandboxes | harness/control plane 与 compute sandbox 分离，支持 commands/files/packages/ports/snapshots/resumable state。 | SDK 支持 multi-agent handoffs，但 sandbox 仍是 provider workspace。 | AKernel 可进一步把 sandbox compute 变成可调度 OS process。https://developers.openai.com/api/docs/guides/agents/sandboxes |
| 2026 | Anthropic Managed Agents self-hosted sandboxes | agent loop 托管，tool execution 放到客户自控 sandbox/provider。 | BYOC 执行边界强，orchestrator 仍在平台。 | 支撑企业执行 perimeter 动机。https://claude.com/blog/claude-managed-agents-updates |
| 2026 | Microsoft MXC | policy-driven execution containers，面向 Windows/WSL/macOS/Linux agent containment。 | endpoint/local containment 方向。 | 与 policy framing 相关，但不是 datacenter OS。https://blogs.windows.com/windowsdeveloper/2026/06/02/windows-platform-security-for-ai-agents/ |

关键结论：AI sandbox 正在快速商品化和云原生化。尤其 GKE Agent Sandbox、Azure Sandboxes、Cube Sandbox 已经开始处理 cluster/snapshot/policy。AKernel 不能声称“别人都是单机，我们是集群”这么简单；更准确的差异是“别人以 sandbox 为主抽象，AKernel 以 agent process/realm/object/policy/provenance 为 OS 抽象，并用真实 workload goodput 验证”。

## 8. Agent Resource Control / Checkpoint / Safety / Provenance

| 时间 | 来源 | 工作 | 关键数据/贡献 | AKernel 关系 |
|---|---|---|---|---|
| 2026-02 | arXiv | AgentCgroup: Understanding and Controlling OS Resources of AI Agents | 144 SWE-rebench task；发现 OS-level execution 占 56-74% latency，memory 是 concurrency bottleneck，tool-call memory spike 最高 15.4x；提出 tool-call-aligned cgroup + eBPF/sched_ext/memcg_bpf_ops。 | 最直接支持 AKernel 的 resource-control 动机。但它主要是单 sandbox/session 内 resource dynamics，不是 agent cloud OS。https://arxiv.org/abs/2602.09345 |
| 2026-04 | arXiv | Crab: A Semantics-Aware Checkpoint/Restore Runtime for Agent Sandboxes | 指出 agent-OS semantic gap；超过 75% agent turns 没有 recovery-relevant OS state；eBPF inspector 决定 checkpoint granularity；checkpoint traffic 最多降 87%，fault-free overhead 1.9%。 | 强力支持“agent turn/tool-call boundary 应进入 OS state lifecycle”。AKernel 可把 Crab 作为 state plane 近邻。https://arxiv.org/abs/2604.28138 |
| 2026-05 | arXiv | AgentTrust | 在 agent 和 tools 之间做 runtime safety interception，allow/warn/block/review，处理 unsafe shell/HTTP/DB/file action。 | 支撑 AKernel policy plane，但它不是 scheduler/runtime/cloud OS。https://arxiv.org/abs/2605.04785 |
| 2026-06 | arXiv | From Agent Traces to Trust | execution provenance/evidence tracing survey，覆盖 tool-use provenance、memory lineage、observability、audit、failure diagnosis。 | 支撑 AKernel provenance 设计，尤其是把 artifact、tool、memory、command 组成 typed graph。https://arxiv.org/abs/2606.04990 |

对 AgentCgroup 的判断：

- 对：它的核心实验是 benchmark task 上的 sandboxed coding agent resource dynamics，基本工作单元是 single agent/sandbox/session。
- 但要更精确：它确实讨论多租户 cloud environments 和资源隔离，所以不能简单说它完全不关心云；它关心的是单 agent execution 的 OS control granularity，而不是多 agent cluster OS。
- AKernel 可以把它作为 motivation 中最强引用之一：AgentCgroup 证明 agent 的 tool-call resource pattern 与传统 container/serverless control 不匹配；AKernel 则把这个观察扩展到 datacenter-scale agent lifecycle。

## 9. Agent Benchmarks / Workload Characterization

### 9.1 Tool/Web/Desktop/Terminal Benchmarks

| 时间 | 来源 | 工作 | Workload 特征 | 是否有 OS resource trace | AKernel 关系 |
|---|---|---|---|---|---|
| 2023 | EMNLP | API-Bank | 73 API tools，多轮 API planning/retrieval/execution。 | 无，主要是 API 调用日志。 | tool gateway/API policy baseline。https://arxiv.org/abs/2304.08244 |
| 2023/2024 | ICLR | ToolLLM / ToolBench | 16k+ real-world APIs，ToolEval。 | 无 OS trace。 | 海量异构工具调用压力。https://arxiv.org/abs/2307.16789 |
| 2023/2024 | ICLR | WebArena | 自托管电商、论坛、GitLab-like、CMS，812 long-horizon web tasks。 | 行为轨迹，无资源 trace。 | browser sandbox/state reset/object capture。https://arxiv.org/abs/2307.13854 |
| 2023/2024 | ICLR | AgentBench | 8 个交互环境，评估 LLM as agents。 | 行为轨迹，无资源 trace。 | 综合 agent task suite。https://arxiv.org/abs/2308.03688 |
| 2023/2024 | ICLR | GAIA | 466 个真实助理任务，推理、多模态、web browsing、tool use。 | 无 OS trace。 | 支撑 agent 不是单次推理。https://arxiv.org/abs/2311.12983 |
| 2024 | ACL/arXiv | VisualWebArena | 视觉 grounded web tasks。 | 行为轨迹，无 OS trace。 | browser screenshot/DOM/action objects。https://arxiv.org/abs/2401.13649 |
| 2024 | ECCV | OmniACT | desktop + web screenshot 到 executable action script。 | action script，无 OS trace。 | GUI as API，要求可回放和环境隔离。https://arxiv.org/abs/2402.17553 |
| 2024 | ICML/NeurIPS | WorkArena / WorkArena++ | ServiceNow enterprise workflow，33 到 682 个知识工作任务。 | 行为轨迹，无资源 trace。 | 企业 SaaS agent workload。https://arxiv.org/abs/2403.07718 |
| 2024 | NeurIPS | OSWorld | 真实 OS/VM 中 369 个 computer-use tasks。 | VM 级执行，公开资源 trace 不充分。 | 与 Computer Use 最直接；AKernel 主张从 Computer Use 到 Datacenter Use。https://arxiv.org/abs/2404.07972 |
| 2024/2025 | ICML | Windows Agent Arena | Windows OS 150+ tasks，Azure 上可并行全量评测。 | 行为轨迹，无公开 OS resource trace。 | 支撑大规模并发 environment pool。https://arxiv.org/abs/2409.08264 |
| 2024/2025 | ICLR | tau-bench | 用户模拟器 + tool agent + domain policy，pass^k reliability。 | 对话/tool/db state，无 OS trace。 | policy/session/reliability 指标。https://arxiv.org/abs/2406.12045 |
| 2026 | arXiv | Terminal-Bench 2.0 | 89 个 hard terminal tasks，每题独立环境、人工解法、测试验证。 | command transcript，通常无 OS trace。 | shell sandbox + terminal workload benchmark。https://arxiv.org/abs/2601.11868 |

### 9.2 Coding / ML / Cyber Agent Workloads

| 时间 | 来源 | 工作 | Workload 特征 | 是否有 OS resource trace | AKernel 关系 |
|---|---|---|---|---|---|
| 2023/2024 | ICLR | SWE-bench | 2294 GitHub issue/PR tasks，patch + tests。 | patch/test logs，无 OS trace。 | coding agent benchmark 核心。https://arxiv.org/abs/2310.06770 |
| 2024 | NeurIPS | SWE-agent | Agent-computer interface，browse/edit/test。 | command/action trajectory，无 OS trace。 | 支持 AKernel tool/syscall ABI 叙事。https://arxiv.org/abs/2405.15793 |
| 2024 | OpenAI/SWE-bench | SWE-bench Verified | 500 人工过滤任务。 | patch/test logs。 | 更可靠 evaluation set。https://www.swebench.com/verified.html |
| 2024/2025 | ICLR | SWE-bench Multimodal | 617 JS/UI/visual bug tasks。 | patch/test/screenshot。 | 多模态 coding artifacts。https://arxiv.org/abs/2410.03859 |
| 2024/2025 | ICLR | MLE-bench | 75 Kaggle ML competitions。 | experiment logs/artifacts，无统一资源 trace。 | GPU/CPU/storage 长任务调度。https://arxiv.org/abs/2410.07095 |
| 2024 | arXiv | Cybench / EnIGMA | CTF/cyber tasks，bash environment。 | command transcript。 | 强隔离和危险工具 policy。https://arxiv.org/abs/2408.08926 |
| 2025 | NeurIPS D&B | SWE-rebench | 21k+ interactive Python SWE tasks，面向 RL 与去污染评测。 | patch/test logs，无 OS trace。 | 大规模 rollout/training environment。https://arxiv.org/abs/2505.20411 |
| 2025 | NeurIPS D&B | SWE-bench-Live | 自动更新、多语言、多 OS SWE tasks。 | patch/test logs。 | continuous benchmark/environment refresh。https://arxiv.org/abs/2505.23419 |

### 9.3 Agent Workload Trace and Characterization

| 时间 | 来源 | 工作 | 关键数据/发现 | AKernel 关系 |
|---|---|---|---|---|
| 2025 | MLSys | AIOpsLab | 部署 microservice cloud env、注入 faults、生成 workload、导出 telemetry。 | 最贴近 AKernel 的 cloud ops agent benchmark，有 telemetry 和真实系统状态。https://arxiv.org/abs/2501.06706 |
| 2026-02 | arXiv | AgentCgroup | SWE-rebench 上 144 SWE tasks，OS-level CPU/mem/I/O/resource dynamics。 | 资源控制最直接证据。https://arxiv.org/abs/2602.09345 |
| 2026-05 | arXiv | Agentic AI Workload Characteristics | ReAct agents 的 LLM serving + tool execution tracing；context caching 后更 decode-dominated；tool behavior 有 temporal structure。 | 支持 AKernel 联合管理 LLM context/KV-cache、tool execution、session state。https://arxiv.org/abs/2605.26297 |
| 2026-06 | arXiv | TraceLab | 4300 coding-agent sessions，约 350k LLM steps、430k tool calls，Claude Code/Codex 日常使用 trace。 | 可作为 AKernel workload model 和 trace-driven evaluation 的关键公开数据源。https://arxiv.org/abs/2606.30560 |

关键结论：主流 agent benchmark 大多只有行为轨迹、tool calls、patch/test logs，而不是 OS-level CPU/mem/I/O/network/object locality trace。AgentCgroup、AIOpsLab、Agentic AI Workload Characteristics、TraceLab 是目前最接近 AKernel motivation 的公开证据，但仍没有给出 datacenter-scale agent OS。

## 10. Multi-Agent Systems / Agent OS / Agentic RL

| 时间 | 来源 | 工作 | Workload/系统特征 | AKernel 关系 |
|---|---|---|---|---|
| 2023 | NeurIPS | CAMEL | role-playing communicative agents。 | 早期多 agent cooperation workload。https://arxiv.org/abs/2303.17760 |
| 2023/2024 | ACL | ChatDev | CEO/CTO/programmer/tester 多角色软件开发。 | agent process graph、artifact handoff、workflow stage。https://arxiv.org/abs/2307.07924 |
| 2023 | Microsoft | AutoGen | 可编程 multi-agent conversation framework。 | AKernel 可把 conversation + tool call 视为 runtime scheduling primitive。https://arxiv.org/abs/2308.08155 |
| 2023/2024 | ICLR | MetaGPT | SOP + assembly-line 多 agent 软件开发。 | 角色隔离、中间产物、workflow-as-code。https://arxiv.org/abs/2308.00352 |
| 2023 | arXiv | AgentVerse | 动态组队和多 agent collaboration。 | group scheduling/shared object/blackboard。https://arxiv.org/abs/2308.10848 |
| 2024 | arXiv/OpenReview | AIOS | LLM Agent Operating System，kernel + SDK + memory/storage/tool/access control/scheduling/context switch。 | 直接相关但更像 agent framework kernel；AKernel 要强调 datacenter OS/resource/sandbox/object locality。https://arxiv.org/abs/2403.16971 |
| 2024 | Microsoft | Magentic-One | Orchestrator + Web/File/Code 等专家 agent。 | 对应 supervisor + worker pool。https://arxiv.org/abs/2411.04468 |
| 2024/2025 | TMLR | BrowserGym | 统一 Gym API 封装 WebArena/WorkArena/MiniWoB。 | reproducible env lifecycle。https://arxiv.org/abs/2412.05467 |
| 2025 | ACL | AgentGym | 14 interactive environments，concurrent exploration，AgentTraj/AgentEval。 | agent RL environment pool。https://arxiv.org/abs/2406.04151 |
| 2025 | arXiv | RAGEN | multi-turn stochastic env 中训练 reasoning agents。 | rollout drift 和可回放环境。https://arxiv.org/abs/2504.20073 |
| 2025 | ACL | MultiAgentBench | star/chain/tree/graph topologies，collaboration/competition KPI。 | 评估 AKernel realm 内 agent topology。https://arxiv.org/abs/2503.01935 |
| 2025 | arXiv | MOYA / Autonomous CloudOps | CloudOps multi-agent framework，集成内部/外部系统和 human control。 | 与 AKernel Datacenter Use 目标最贴近。https://arxiv.org/abs/2501.08243 |
| 2025 | arXiv | Murakkab | Resource-efficient agentic workflow orchestration，解耦 workflow components、model/hardware choices，优化 cost/latency/energy/SLO。 | 这是 AKernel 必须跟踪的 agentic workflow cloud platform 近邻；更偏 model/tool workflow orchestration，AKernel 偏 OS/sandbox/state/object/policy。https://arxiv.org/abs/2508.18298 |
| 2025 | arXiv | Towards Resource-Efficient Compound AI Systems | compound AI systems 的 resource utilization 问题，指出 orchestration 与 resource management 脱节。 | 支撑 AKernel “agent logic and resource management should not be disconnected”。https://arxiv.org/abs/2501.16634 |

关键结论：multi-agent framework 和 Agent OS 方向已经存在，但大多是 application/runtime/framework 层：conversation、roles、tools、memory、LLM scheduling。AKernel 的空间在更底层：把这些 agent execution units 作为云端 OS 进程和对象来管理。

## 11. 差距矩阵

| 系统/方向 | Sandbox primitive | Multi-node / cluster | Stateful lifecycle | Object locality | Policy/capability | Provenance/audit | Workload trace | AKernel 差异 |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| E2B/Modal/Daytona | 强 | 平台隐含 | 中到强 | 弱 | 中 | 弱 | 弱 | sandbox API 已商品化，缺 agent OS abstraction |
| OpenSandbox/Cube | 强 | 中到强 | 中 | 弱 | 中到强 | 弱 | 弱 | 可作为 runtime substrate/baseline |
| GKE Agent Sandbox | 强 | 强 | 中 | 弱 | K8s 级 | 弱 | 产品指标 | K8s primitive，而非 agent realm/object OS |
| Azure Sandboxes | 强 | 中到强 | 强 | 弱 | 强 | 弱 | 产品指标 | sandbox group，而非 agent-native cloud OS |
| SigmaOS | 中 | 强 | serverless/microservice | object API | 弱 | 弱 | 系统实验 | cloud OS 近邻，但非 agent-native |
| YuanRong | 中 | 强 | serverless/distributed app | data system | 弱到中 | 弱 | 生产数据 | serverless kernel 近邻，需 agent 改造 |
| AFaaS/cold start | 强 | 中 | start/fork/clone | image locality | 弱 | 弱 | 生产 cold start | 节点侧 cold start，不是 agent lifecycle OS |
| Cloudburst/SONIC/Pocket | 弱 | 中 | function state | 强 | 弱 | 弱 | workload 实验 | stateful serverless，不含 agent side effects |
| AgentCgroup | 依赖 sandbox | 多租户资源控制 | tool-call resource | 弱 | resource policy | 弱 | 有 OS trace | 单 agent/session resource control |
| Crab | 依赖 sandbox | host/co-located | checkpoint/restore 强 | 弱 | 弱 | recovery semantics | 有实验 | state plane 机制，不是 cloud OS |
| AIOS/AutoGen/MetaGPT | 弱 | framework 层 | memory/tool/runtime | 弱 | 中 | 中 | 行为轨迹 | agent framework，不是 OS substrate |
| AKernel 目标 | 强 | 强 | 强 | 强 | 强 | 强 | 需要真实 trace | 统一 agent process/realm/object/policy/provenance |

## 12. AKernel 应该如何回答“创新性”

### 12.1 可以成立的强 claim

可以写：

> Existing AI sandbox systems expose useful per-agent execution primitives, but they leave datacenter-level state lifecycle, object locality, policy-carrying execution, and agent-level goodput to applications. AKernel introduces Agent Processes, Agent Realms, and Agent Objects as OS-level abstractions for stateful, artifact-heavy, policy-constrained agent execution.

这个 claim 的优点：

- 主动承认 sandbox API 已经 commodity 化。
- 不和 Cube/GKE/Azure 争“谁启动更快”。
- 把贡献推到 agent-specific OS semantics。
- 可以和 SigmaOS/YuanRong 清楚区分。

### 12.2 不应使用的弱 claim

不要写：

- “我们是第一个 agent sandbox。”
- “我们是第一个支持 snapshot/fork 的 agent platform。”
- “我们比 E2B/Modal 更快。”
- “我们把 Kubernetes 用在 agent 上，所以是 agent OS。”
- “AgentCgroup 只做单实例，所以我们做多实例就足够创新。”

这些说法都容易被反驳。GKE Agent Sandbox、Azure SandboxGroup、Cube Sandbox、OpenSandbox 已经把 sandbox 做到 K8s/cluster/microVM/snapshot/policy；SigmaOS/YuanRong 已经把 cloud OS/serverless kernel 做得很强。

### 12.3 最有说服力的新 insight

建议把 AKernel 的新 insight 写成：

> The scheduling unit of datacenter agents is not a container, VM, function, or LLM request. It is a stateful, policy-carrying, artifact-producing Agent Process within an Agent Realm. Its performance depends on the joint locality of images, checkpoints, object artifacts, package caches, model/context state, and policy setup.

如果 trace 能证明以下事实，就很有 OSDI/SOSP 味道：

- sandbox boot 只占端到端 useful start 的一小段。
- idle/wait 占大量 wall-clock，checkpoint-on-idle 能省资源。
- object/artifact reuse 有强 locality，object-aware scheduling 能降跨节点流量。
- branch/fork/retry 是 agentic RL/coding/eval 的常见模式。
- policy/provenance 不是附加日志，而是安全恢复、审计和企业部署的必要执行语义。
- 成功任务数/成本/SLO 比 raw throughput 更能反映 agent workload。

## 13. Evaluation 建议

### 13.1 Baselines

至少需要这些 baseline：

| Baseline | 用途 |
|---|---|
| Kubernetes + containerd/gVisor/Kata | 传统云原生 agent sandbox baseline |
| openYuanRong baseline | 证明 AKernel 不是直接继承 YuanRong 就够了 |
| E2B-like single-sandbox baseline | 对比单 sandbox API 模型 |
| OpenSandbox/Cube-like local baseline | 对比开源 AI sandbox runtime |
| AKernel minus checkpoint/restore | 证明 lifecycle/state plane |
| AKernel minus object-local scheduling | 证明 Agent Object/locality |
| AKernel minus lazy image/P2P | 证明 image plane |
| AKernel minus policy/provenance | 证明安全语义 overhead 和收益 |

如果无法公平测闭源 E2B/Modal/Daytona，就实现等价 API baseline，并明确说明闭源平台无法完全公平对比。

### 13.2 Workloads

建议覆盖 5 类：

| Workload | 可用来源 | 展示重点 |
|---|---|---|
| Coding agent | SWE-bench Verified, SWE-rebench, TraceLab-like internal trace | repo checkout、dependency、test、artifact、retry、tool-call memory spikes |
| Terminal/Ops agent | Terminal-Bench, AIOpsLab, internal ops tasks | shell sandbox、network policy、long-running state、audit |
| Agentic RL rollout | SWE-rebench/OSWorld/BrowserGym/内部 RL | burst scale、fork、checkpoint、parallel environment pools |
| Data/ML agent | MLE-bench, Spark/FaaS mixed tasks | data locality、CPU/GPU/storage、object reuse |
| Multi-agent workflow | AutoGen/MetaGPT/MultiAgentBench/内部 swarm | realm、shared object、topology、policy isolation |

### 13.3 Metrics

核心指标应是：

- time-to-useful-action，而不是只报 cold start
- successful tasks/hour
- cost per successful task
- cost-to-solution
- SLO 内完成的正确任务数
- resource-hours saved by checkpoint/hibernate
- checkpoint/restore/fork latency and size
- object cache hit rate
- cross-node traffic
- image/cache working set
- p50/p95/p99 task latency
- policy setup/enforcement overhead
- provenance log size and query/debug latency

### 13.4 必须有的 ablation

| Ablation | 预期回答的问题 |
|---|---|
| no checkpoint/restore | agent idle/wait 是否真的浪费资源，C/R 是否省钱 |
| rootfs-only checkpoint vs full lifecycle | agent state 是否超出文件系统 snapshot |
| no object locality | object-aware scheduling 是否减少跨节点流量/latency |
| no image locality/lazy loading | 大镜像/依赖是否影响 useful start |
| no policy/provenance | 安全语义开销是多少，调试/审计收益是什么 |
| random placement vs AKernel placement | compute-near-state 是否成立 |
| single sandbox baseline vs realm-level scheduling | 多 agent realm 是否是必要抽象 |

## 14. 建议论文写法

### 14.1 Introduction 逻辑

1. Agent 正在从 Computer Use 走向 Datacenter Use。
2. 现有 AI sandbox 已经能提供一台隔离电脑。
3. 真实 agent workload 暴露的问题在 sandbox 之上：stateful、bursty、artifact-heavy、idle-prone、policy-constrained。
4. 单 sandbox/API 无法优化 agent-goodput，因为关键状态分布在 image、workspace、checkpoint、object store、LLM context、tool/network policy 中。
5. AKernel 提出 Agent Process/Realm/Object/Datacenter Function，并实现跨层 agent OS。
6. 用真实 trace 和端到端实验验证。

### 14.2 Related Work 组织方式

不要按大杂烩列。建议按“为什么不够”组织：

1. Cloud OS and distributed execution：SigmaOS、YuanRong、Borg、Omega、Ray、SkyPilot。
2. Serverless state and workflow：Cloudburst、Beldi、Pocket、SONIC、FaaSFlow、Halfmoon。
3. Cold start and sandbox lifecycle：Firecracker、SOCK、SEUSS、Catalyzer、FaaSnap、AFaaS、Dirigent、Pronghorn。
4. AI sandbox platforms：E2B、Modal、Daytona、OpenSandbox、Cube、GKE Agent Sandbox、Azure/Cloudflare Sandboxes。
5. Agent workloads and frameworks：AgentCgroup、TraceLab、Agentic AI Workload Characteristics、OSWorld、Terminal-Bench、SWE-bench、AIOS、AutoGen、MultiAgentBench。

### 14.3 一段可直接改写进论文的定位句

> AKernel is not another sandbox API. It is an agent-native cloud operating system motivated by production traces of stateful, bursty, artifact-heavy, idle-prone, and policy-constrained agent workloads. AKernel makes Agent Processes, Agent Realms, and Agent Objects first-class OS abstractions, and co-designs scheduling, sandbox lifecycle, checkpoint/restore, object locality, image loading, policy enforcement, and provenance to optimize agent-level goodput and cost-to-solution at datacenter scale.

## 15. 风险清单

| 风险 | 为什么危险 | 缓解方式 |
|---|---|---|
| 被认为是 sandbox SDK | E2B/Modal/Daytona/OpenSandbox/Cube 已覆盖很多 API | 主动承认 commodity sandbox，贡献写 agent OS semantics |
| 被认为是 SigmaOS/YuanRong 的 agent workload 版本 | cloud OS/serverless kernel 已有强论文 | 用 trace 证明 agent-specific state/policy/provenance/locality |
| 被认为只是 cold start 优化 | AFaaS/FaaSnap/Catalyzer/SEUSS 已很强 | 使用 time-to-useful-action 和 end-to-end goodput |
| 没有真实 trace | OSDI/SOSP 对 workload claim 要求高 | 至少 1k sessions，最好覆盖 coding/RL/data/ops/multi-agent |
| 无法公平对比闭源产品 | E2B/Modal/Azure/GKE 部分不可复现 | 做等价 local baseline，公开限制 |
| Provenance/policy 只做日志 | 审稿人会认为不是系统核心 | 让 policy/provenance 参与调度、checkpoint、audit、recovery |
| BYOC 变成工程部署展示 | Helm/Terraform 本身不是研究贡献 | 只作为 deployment/experience，主线仍是 agent lifecycle |

## 16. 最小可投稿证据包

如果目标是 OSDI/SOSP，最小证据包建议如下：

1. 真实 agent workload trace，至少 1000 sessions，覆盖 coding、terminal/ops、agentic RL/eval、data/ML、多 agent 中至少 3 类。
2. AI sandbox commodity 对比表，明确 E2B/Modal/Daytona/OpenSandbox/Cube/GKE/Azure 已提供哪些 primitive。
3. Agent workload characterization：duration、active/idle ratio、tool calls、artifact/object size、checkpoint size、object reuse、network/policy events、success/cost。
4. End-to-end agent goodput 实验：successful tasks/hour、cost per successful task、time-to-useful-action。
5. Checkpoint/restore/hibernate/fork 实验：resource-hours saved、restore latency、lost work after failure。
6. Object locality 实验：cache hit、cross-node traffic、object transfer latency、task runtime。
7. Burst scaling 实验：1/100/1k/10k Agent Process create/restore/fork，不只报 ready，也报 useful action。
8. Policy/provenance 实验：overhead、blocked unsafe actions、audit/debug query。
9. Production experience：内部部署、运维、故障案例、AI-assisted debugging，只作为辅助贡献。

## 17. 最终判断

你的直觉是正确的：AgentCgroup 这类工作证明 agent 已经成为 OS 资源管理问题，但它仍主要停在 single-agent/sandbox resource control。AKernel 更有意思的方向是 datacenter-scale agent OS：许多 agent process 在多个 node 上运行、共享和复用 object/artifact/checkpoint/cache，并由 realm 统一管理 lifecycle、policy、provenance 和 goodput。

这符合 OSDI/SOSP 级别创新的潜力，但不是自动成立。论文必须把“多 agent node 组成 agent OS”落到可测量的新抽象和机制上：

- Agent Process 不是 container：它携带 tool/policy/state/provenance。
- Agent Realm 不是 namespace：它是多 agent workflow 的资源、对象、权限和成功率边界。
- Agent Object 不是普通文件：它是调度、恢复、复用和审计的系统对象。
- Datacenter Function 不是 FaaS：它展开为带状态、带策略、带对象依赖的分布式 agent execution。

只要 AKernel 能用真实 trace 和端到端实验证明这些抽象带来 agent-goodput/cost/locality/security 的系统收益，这个方向是有机会达到 OSDI/SOSP 要求的。反过来，如果没有 trace，只展示 sandbox、lazy image、P2P、checkpoint、Helm/Terraform 的组合，创新性会明显不足。

