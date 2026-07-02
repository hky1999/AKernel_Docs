# AgentCgroup 与 Agent Workload 相关论文/资料 Survey

调研完成时间：2026-07-02 04:20:44 UTC

主题：围绕 `AgentCgroup` 的 OS/resource/sandbox/serving 相关工作，以及 agent workload/benchmark/trace 的系统性调研。范围包括 OSDI/SOSP/NSDI/USENIX ATC/EuroSys/ASPLOS/FAST/SIGCOMM/SoCC/MLSys/HotOS/HotCloud/NeurIPS/ICLR/ICML/ACL/EMNLP/AAAI/CVPR/ECCV/TMLR/OpenReview/arXiv，以及 industry blogs/docs/GitHub。

本次调研使用了 3 个并行 subagent：

- OS/systems subagent：AgentCgroup 相近的 OS、sandbox、resource control、agent serving、compound AI systems。
- AI/ML benchmark subagent：agent benchmark、workload、真实 trace、multi-agent、GUI/web/coding/cloud ops benchmark。
- industry/blog subagent：OpenAI、Anthropic、Google/GKE、Azure、Cloudflare、E2B、Modal、Daytona、OpenSandbox、Cube、LangGraph/LangSmith、Docker 等工程资料。

## 0. 总结

`AgentCgroup` 处在一个正在快速形成的新方向里：**agent workload 不再只是 LLM serving workload，而是 LLM 调用、tool execution、sandbox process tree、filesystem diff、network/API side effect、checkpoint/rollback、human-paced gap、multi-agent workflow 共同组成的 OS workload**。

截至 2026-07-02，相关工作大致分成 7 类：

1. **OS/resource control for agents**：AgentCgroup、Sandlock、Grimlock、Rethinking OS Interfaces、AIOS、AgenticOS workshop。
2. **agent checkpoint/rollback/state exploration**：Fork-Explore-Commit、Crab、DeltaBox、ACRFence、LangGraph checkpoint/HITL。
3. **agentic serving / workflow scheduling**：Parrot、AgentServe、Helium、Murakkab、Cortex、Continuum、Kairos、Agentix/Autellix、Pie、CPU-centric agentic execution。
4. **coding-agent and terminal-agent traces**：TraceLab、SWE-chat、SWE-bench/SWE-rebench/SWE-bench-Live、Terminal-Bench。
5. **computer-use/web/GUI benchmarks**：OSWorld、WebArena、WorkArena、Mind2Web、WebLINX、VisualWebArena、AndroidWorld、BrowserGym。
6. **cloud ops / enterprise agent workloads**：AIOpsLab、Cloud-OpsBench、ITBench、Microservice Failure Diagnosis / AgentOps datasets、TheAgentCompany。
7. **industry execution platforms and observability**：OpenAI Codex/Agents tracing、Anthropic Managed Agents/containment/Claude Code sandboxing、E2B metrics、Daytona telemetry/RL infra bench、Cloudflare Sandboxes、GKE Agent Sandbox、Cube Sandbox、OpenSandbox、Modal、Docker sandboxes。

最接近 `AgentCgroup` 的论文不是传统 serverless/cgroup 论文，而是 2026 年这一批 **OS-for-agent** 论文：

- AgentCgroup：tool-call-aligned cgroup/eBPF resource control。
- Fork, Explore, Commit：agentic exploration 的 OS branch context。
- Crab：turn-level semantics-aware checkpoint/restore。
- DeltaBox：millisecond-level sandbox checkpoint/rollback。
- Sandlock：unprivileged Linux primitives for agent code confinement。
- ACRFence：agent checkpoint-restore 的 semantic rollback attack。
- Rethinking OS Interfaces for LLM Agents：给 computer-use agents 的 declarative OS interface。
- Grimlock：eBPF/attested channel guard for high-agency systems。

这些工作共同说明：agent workload 的关键边界不是传统 `process/container/request`，而是 `turn/tool-call/branch/session/agent workflow/object side effect`。

## 1. AgentCgroup 作为中心工作

### AgentCgroup: Understanding and Controlling OS Resources of AI Agents

- 时间：2026-02-10 arXiv；AgenticOS @ ASPLOS 2026 workshop paper
- URL：https://arxiv.org/abs/2602.09345
- 本地 PDF：`References/Zheng 等 - 2026 - AgentCgroup Understanding and Controlling OS Resources of AI Agents.pdf`
- 对象：sandboxed AI coding agents；SWE-rebench 中 144 个 software engineering tasks；两个 LLM。
- 关键发现：
  - OS-level execution，包括 tool calls、container initialization、agent initialization，占端到端 latency 的 56-74%。
  - memory 而非 CPU 是多 agent 并发下的主要 bottleneck。
  - memory spike 由 tool-call 驱动，峰值可达 peak-to-average 15.4x。
  - resource demand 跨 task、run、model 都高度不可预测。
  - 传统 resource control 有三个 mismatch：
    - granularity mismatch：container-level policies vs tool-call-level dynamics。
    - responsiveness mismatch：user-space reaction vs sub-second unpredictable bursts。
    - adaptability mismatch：history-based prediction vs nondeterministic stateful execution。
- 机制：AgentCgroup 使用 intent-driven eBPF-based resource controller，将 cgroup hierarchy 对齐到 tool-call boundary，结合 `sched_ext` 和 `memcg_bpf_ops` 做 in-kernel enforcement。
- 与 AKernel 的关系：
  - 这是 AKernel workload motivation 的强引用：agent workload 不是传统 container/serverless/microservice/batch。
  - 它仍主要在单 agent/session/sandbox 内刻画 resource dynamics，最多扩展到 multi-tenant isolation。
  - AKernel 可以把 AgentCgroup 扩展到 datacenter OS：per-realm/per-agent/per-tool/per-object resource accounting, scheduling, policy, checkpoint, provenance。

## 2. OS / Runtime / Sandbox 方向：AgentCgroup 的近邻

### 2.1 2024

| 年份 | 来源 | 工作 | URL | Workload/对象 | 关键点 | 与 AgentCgroup / AKernel 的关系 |
|---|---|---|---|---|---|---|
| 2024 | arXiv / OpenReview | AIOS: LLM Agent Operating System | https://arxiv.org/abs/2403.16971 | LLM agent runtime，agent scheduling, context, memory, storage, tool, access control | 提出 AIOS kernel，把 LLM/tool/memory/storage/access-control 从 agent app 中抽离，实验显示最高 2.1x faster execution | 上层 agent OS 蓝图；缺真实 OS resource trace 和 cgroup/eBPF enforcement。AKernel 可把它具体化为 cloud OS。 |
| 2024 | OSDI | Parrot: Efficient Serving of LLM-based Applications with Semantic Variable | https://www.usenix.org/conference/osdi24/presentation/lin-chaofan | 多租户 LLM apps / agent workflow 中的多 LLM request DAG | Semantic Variable 暴露 app-level 语义，服务端做跨请求数据流分析和端到端优化 | 与 AgentCgroup 一样反对 opaque request；Parrot 管 LLM semantic/dataflow，AgentCgroup 管 tool/OS resource。 |
| 2024 | OSDI | Llumnix: Dynamic Scheduling for Large Language Model Serving | https://www.usenix.org/conference/osdi24/presentation/sun-biao | LLM serving 中异构、不可预测请求 | 动态调度处理 LLM request duration/resource unpredictability | 背景是 LLM engine；AgentCgroup 把不可预测性推进到 sandbox/tool 层。 |
| 2024 | OSDI | ServerlessLLM: Low-Latency Serverless Inference for LLMs | https://www.usenix.org/conference/osdi24/presentation/fu | Serverless LLM inference cold start | 优化 model loading/serverless inference cold start | AKernel 可把 model cold start、sandbox cold start、workspace restore 放进统一 useful-start path。 |
| 2024 | arXiv / dataset | BurstGPT: Towards Efficient and Reliable LLM Serving | https://arxiv.org/abs/2401.17644 | Azure OpenAI GPT-3.5/GPT-4 serving trace；10.31M traces over 213 days | 真实 LLM serving trace，分析 burstiness、conversation pattern、response length、failures | 不是 agent trace，但提供“真实 trace 驱动系统设计”的范式。AKernel 应有 agent OS trace。 |

### 2.2 2025

| 年份 | 来源 | 工作 | URL | Workload/对象 | 关键点 | 与 AgentCgroup / AKernel 的关系 |
|---|---|---|---|---|---|---|
| 2025 | HotOS / Microsoft Research | Towards Resource-Efficient Compound AI Systems | https://www.microsoft.com/en-us/research/publication/towards-resource-efficient-compound-ai-systems/ | Compound AI workflows，models/retrievers/tools 多组件 | 指出 inefficiency 来自 application logic 与 execution details 耦合、orchestration 与 resource management 脱节 | AgentCgroup 是底层 OS resource answer；AKernel 可成为 compound/agentic cloud OS。 |
| 2025 | arXiv | Murakkab: Resource-Efficient Agentic Workflow Orchestration in Cloud Platforms | https://arxiv.org/abs/2508.18298 | Multi-tenant agentic workflows；models/tools/hardware/SLOs | 声明式 workflow abstraction，profile-guided optimizer + adaptive runtime；报告 GPU usage 最高降 2.8x、energy 3.7x、cost 4.3x | Cloud platform 上层近邻。AgentCgroup 可作为底层 actuator；AKernel 可向上接 declarative workflow，向下接 sandbox/cgroup/object。 |
| 2025 | arXiv | Cortex: Workflow-Aware Resource Pooling and Scheduling for Agentic Serving | https://arxiv.org/abs/2510.14126 | Agentic workflow serving；stage-level pools | stage isolation，为 agent workflow 不同 stage 分配专属 resource pool，改善 interference、KV cache、predictability | 与 AgentCgroup 都强调 stage/tool granularity；AKernel 可把 stage/tool-class 做成 scheduling namespace。 |
| 2025 | arXiv / SOSP | Pie: A Programmable Serving System for Emerging LLM Applications | https://arxiv.org/abs/2510.24051 | Agentic workflows、custom reasoning、multimodal serving | 把 monolithic token loop 拆成 handlers，用 Wasm sandbox 执行 inferlets；agentic workflows 上提升 1.3x-3.4x | 说明 agent serving 需要可编程 substrate；AKernel 可用 eBPF/Wasm 式 policy 扩展 agent OS。 |
| 2025 | arXiv | Continuum: Efficient and Robust Multi-Turn LLM Agent Scheduling with KV Cache TTL | https://arxiv.org/abs/2511.02230 | ReAct/multi-turn agents；tool-call gaps 中的 KV cache | 工具调用间隙不等同于 conversation idle；选择性保留 KV cache，降低 agent job completion time | 与 AgentCgroup 互补：Continuum 管 LLM/KV，AgentCgroup 管 OS/tool。AKernel 应联合二者。 |
| 2025 | arXiv | Kairos: Low-latency Multi-Agent Serving with Shared LLMs | https://arxiv.org/abs/2508.06948 | 多 agent 共享 LLM | workflow-aware priority scheduler + memory-aware dispatcher，降低 multi-agent app end-to-end latency | LLM serving 层的多 agent interference；AKernel 要处理 LLM 与 sandbox host 双重干扰。 |
| 2025 | arXiv | A CPU-Centric Perspective on Agentic AI Execution | https://arxiv.org/abs/2511.00739 | Haystack/LangChain/Toolformer/mini-SWE-agent 等 CPU-heavy tool workloads | 指出 agentic AI 强依赖 CPU-GPU heterogeneous systems，许多 tool/orchestration 在 CPU 上运行 | 强支撑 AgentCgroup：agent 不是纯 GPU/LLM workload。AKernel 应把 CPU/mem/IO/process/network 作为一等资源。 |
| 2025 | NeurIPS D&B / arXiv | SWE-rebench | https://arxiv.org/abs/2505.20411 | 21k+ interactive Python SWE tasks | 自动构建 decontaminated SWE benchmark，适合 RL 和大规模 SWE agent 评测 | AgentCgroup 使用其子集；AKernel 可做大规模 coding-agent workload。 |

### 2.3 2026

| 年份 | 来源 | 工作 | URL | Workload/对象 | 关键点 | 与 AgentCgroup / AKernel 的关系 |
|---|---|---|---|---|---|---|
| 2026 | AgenticOS / arXiv | AgentCgroup | https://arxiv.org/abs/2602.09345 | 144 SWE-rebench tasks，sandboxed coding agents | tool-call-level OS resource dynamics；cgroup/eBPF/sched_ext/memcg enforcement | 核心基线。AKernel 可扩展到 cloud-scale agent process/realm controller。 |
| 2026 | arXiv / AgenticOS | Fork, Explore, Commit: OS Primitives for Agentic Exploration | https://arxiv.org/abs/2602.08199 | 并行探索型 agents；filesystem/process state | 提出 branch context：COW FS view + process groups；fork/explore/commit/abort；first-commit-wins；sub-350us branch creation | 与 AgentCgroup 互补：一个管 exploration state，一个管 resource。AKernel 应提供 branch/transaction workspace。 |
| 2026 | arXiv | ACRFence: Preventing Semantic Rollback Attacks in Agent Checkpoint-Restore | https://arxiv.org/abs/2603.20625 | Agent C/R 后不可逆 external tool effects | 定义 semantic rollback attacks：LLM restore 后可能重新合成不同请求，外部服务认为是新操作；提出 replay-or-fork mitigation | 对 AKernel state plane 很关键：rollback 需要记录 tool intent/effect，区分可逆/不可逆操作。 |
| 2026 | arXiv | AgentServe: Algorithm-System Co-Design for Efficient Agentic AI Serving | https://arxiv.org/abs/2603.10342 | 多 agent short reasoning-action loops；single GPU SLM serving | 区分 cold prefills、resume prefills、short decodes；CUDA Green Context slots；TTFT 最高 2.8x，TPOT 最高 2.7x | LLM serving 层的 AgentCgroup 近邻。AKernel 应把 model-side resource 与 sandbox/tool resource 联动。 |
| 2026 | SIGMOD / arXiv | Helium: Efficient LLM Serving for Agentic Workflows | https://arxiv.org/abs/2603.16104 | Agentic workflows as query plans；LLM invocations as operators | 从 data systems 视角做 proactive caching / cache-aware scheduling；最高 1.56x speedup | 说明 agent workflow 要跨调用优化；AKernel 可借鉴 query optimizer，把 execution plan 与 resource plan 联合优化。 |
| 2026 | arXiv | Crab: Semantics-Aware Checkpoint/Restore Runtime for Agent Sandboxes | https://arxiv.org/abs/2604.28138 | Sandboxed containers/microVMs；filesystem/process/runtime artifacts | agent-framework 看 tool calls，OS 看 state changes，双方存在 semantic gap；75% turns 无 recovery-relevant checkpoint；traffic 最多降 87%，overhead 1.9% | 极强相关：tool-call boundary + OS effect + C/R policy。AKernel state plane 应吸收这个 insight。 |
| 2026 | arXiv | DeltaBox: Millisecond-Level Sandbox Checkpoint/Rollback | https://arxiv.org/abs/2605.22781 | SWE-bench、RL micro-benchmarks；stateful AI agents | DeltaState/DeltaFS/DeltaCR，change-based transactional C/R；约 14ms checkpoint、5ms rollback | 与 AgentCgroup 互补：为 branch/exploration/RL rollout 提供快速 state primitive。 |
| 2026 | arXiv | Sandlock: Confining AI Agent Code with Unprivileged Linux Primitives | https://arxiv.org/abs/2605.26298 | AI-agent workstation；filesystem/network/IPC/syscall policy | 使用 Landlock、seccomp-bpf、seccomp notification，无 root/cgroups/images/mandatory namespaces；约 5ms startup overhead | 与 AgentCgroup 都用 Linux primitive；Sandlock 偏安全隔离，AgentCgroup 偏资源控制。AKernel 应合并 security + resource。 |
| 2026 | AgenticOS / arXiv | Rethinking OS Interfaces for LLM Agents / Declarative Model Interface | https://arxiv.org/abs/2510.04607 | Computer-use agents；GUI vs declarative OS interface | 将 GUI 转为 `access/state/observation` declarative primitives；Office tasks 上成功率 +67%，interaction steps -43.5% | 与 AgentCgroup 共享“semantic gap”问题；AKernel 应提供 declarative tool/resource API。 |
| 2026 | AgenticOS / preprint | Skills Are the New Apps: Now It’s Time for Skill OS | https://www.preprints.org/manuscript/202602.1096 | Agent skills as execution artifacts | 主张 OS 管理 skill caching、execution env、global skill management、failure handling | AgentCgroup 可按 skill/tool 学习 resource profile；AKernel 可把 skill 作为对象/调度单位。 |
| 2026 | AgenticOS workshop | Grimlock: Guarding High-Agency Systems with eBPF and Attested Channels | https://os-for-agent.github.io/asplos-2026.html | High-agency systems；eBPF + attested channels | workshop paper；面向 high-agency agents 的 OS guard | 与 AgentCgroup 同属 eBPF/OS enforcement 方向；可作为 policy/security plane related work。 |
| 2026 | AgenticOS workshop | Toward LLM-Driven Rule Generation for Enforcement Systems: WAF | https://os-for-agent.github.io/asplos-2026.html | LLM-generated enforcement rules | 探索让 LLM 生成 enforcement rules | 与 AKernel policy plane 相关，但应谨慎：LLM-generated policy 需要验证。 |
| 2026 | AgenticOS workshop | Fuyun: Bridging the Semantic Gap in Serverless Resource Provisioning via LLM Agents | https://os-for-agent.github.io/asplos-2026.html | serverless resource provisioning | 用 LLM agents 弥合 provisioning semantic gap | 与 AKernel deployment/resource plane 有关。 |
| 2026 | NSDI / official page | Agentix / Autellix: Efficient Serving Engine for LLM Agents as General Programs | https://www.usenix.org/conference/nsdi26/presentation/luo | LLM agents as general programs | program-level context 调度 LLM calls，报告同 latency 下 throughput 4-15x | 支持“agent program/session 是 serving unit”。AKernel 应有 per-program/session control loop。 |
| 2026 | OSDI / arXiv | Murakkab | https://arxiv.org/abs/2508.18298 | agentic workflows in cloud platforms | 见上；OSDI 2026 official URL 可能对 curl 403，但 arXiv 已核验 | 与 AKernel 极近：workflow/cloud/runtime 层；AKernel 需要强调 OS/sandbox/state/object locality。 |
| 2026 | AgenticOS workshop | AgenticOS: Workshop on Operating Systems Design for AI Agents | https://os-for-agent.github.io/asplos-2026.html | OS primitives, isolation, scheduling, observability for agents | workshop 明确提出传统 process/thread/file/socket/resource controllers 不适配 dynamic semantic agent workloads | 证明 AgentCgroup 所在问题已进入 OS 社区议程。AKernel 可定位为 cloud-side Agentic OS。 |

## 3. Agentic Serving / Workflow Scheduling：与 AgentCgroup 互补的系统论文

AgentCgroup 主要刻画 sandbox/tool 层 OS resource dynamics；以下工作主要刻画 LLM serving/workflow orchestration 层。AKernel 如果要写“agent workload”，这两层都要覆盖。

| 年份 | 工作 | URL | 关注点 | 可借鉴 insight |
|---|---|---|---|---|
| 2024 | Parrot | https://www.usenix.org/conference/osdi24/presentation/lin-chaofan | semantic variables for LLM app dataflow | agent runtime 需要暴露 app/semantic structure，opaque RPC 不够。 |
| 2024 | Llumnix | https://www.usenix.org/conference/osdi24/presentation/sun-biao | dynamic LLM serving scheduling | LLM request resource demand 不可预测，agent tool-loop 会放大 tail latency。 |
| 2024 | ServerlessLLM | https://www.usenix.org/conference/osdi24/presentation/fu | serverless LLM cold start | Agent useful-start 包括 model-side cold start 和 sandbox-side restore。 |
| 2025 | Compound AI Systems vision | https://www.microsoft.com/en-us/research/publication/towards-resource-efficient-compound-ai-systems/ | orchestration-resource disconnect | AKernel 的核心可以写成 reconnect agent orchestration and cloud resource management。 |
| 2025/2026 | Murakkab | https://arxiv.org/abs/2508.18298 | declarative agentic workflows + adaptive runtime | AKernel 可作为底层 OS substrate；避免与其只在 workflow optimizer 上竞争。 |
| 2025 | Cortex | https://arxiv.org/abs/2510.14126 | stage isolation/resource pools | tool/stage granularity 比 container granularity 更适合 agent。 |
| 2025 | Pie | https://arxiv.org/abs/2510.24051 | programmable serving handlers / Wasm inferlets | 系统要允许 agentic workloads 自定义 serving logic，但必须 sandbox。 |
| 2025 | Continuum | https://arxiv.org/abs/2511.02230 | KV cache TTL during tool gaps | tool execution gap 是调度机会，不是 idle noise。 |
| 2025 | Kairos | https://arxiv.org/abs/2508.06948 | multi-agent shared LLM orchestration | multi-agent latency/resource dependency 应进 scheduler。 |
| 2025 | CPU-centric agentic execution | https://arxiv.org/abs/2511.00739 | CPU-heavy tools/orchestration bottleneck | agent system 不能只优化 GPU；CPU/mem/IO/network 是主线。 |
| 2026 | AgentServe | https://arxiv.org/abs/2603.10342 | cold/resume prefill + short decode under agent loops | agent serving request types 应分解，而不是统一排队。 |
| 2026 | Helium | https://arxiv.org/abs/2603.16104 | agentic workflows as query plans | 对 AKernel 的启发：Agent Realm/Datacenter Function 可以被 optimizer 处理。 |
| 2026 | Agentix/Autellix | https://www.usenix.org/conference/nsdi26/presentation/luo | agents as general programs | per-program/session scheduling 比 per-request scheduling 更合理。 |

## 4. Agent Workload Trace / Characterization

这些是最能支撑 AKernel motivation 的论文。建议论文中优先引用和复现实验风格。

| 年份 | 来源 | 工作 | URL | Trace / workload | 资源 trace | 与 AgentCgroup / AKernel 的关系 |
|---|---|---|---|---|---|---|
| 2024 | arXiv / dataset | BurstGPT | https://arxiv.org/abs/2401.17644 | 10.31M Azure OpenAI GPT traces over 213 days | LLM serving trace，不含 tool/sandbox OS metrics | 作为真实 trace 方法学参考。 |
| 2025 | MLSys | AIOpsLab | https://arxiv.org/abs/2501.06706 | microservice cloud env, fault injection, workload generation, telemetry export | 有 telemetry，接近系统 trace | AgentOps/cloud ops benchmark，AKernel 高相关。 |
| 2025 | NeurIPS D&B | SWE-rebench | https://arxiv.org/abs/2505.20411 | 21k+ interactive Python SWE tasks | 无 OS resource trace，但适合采集 | AgentCgroup 使用其子集；AKernel 可扩展到大规模 workload。 |
| 2025 | NeurIPS D&B | SWE-bench-Live | https://arxiv.org/abs/2505.23419 | 1319 live-updatable tasks，93 repos，Docker images | 无 OS trace，但有 dedicated Docker image | 支撑 contamination-free, executable, evolving agent workload。 |
| 2026 | arXiv | Terminal-Bench 2.0 | https://arxiv.org/abs/2601.11868 | 89 hard terminal tasks，每题独立环境、tests | 可采集 command/process/file/resource trace | AgentCgroup 最直接 CLI sandbox workload。 |
| 2026 | arXiv | AgentCgroup | https://arxiv.org/abs/2602.09345 | 144 SWE-rebench tasks, 2 LLMs | 有 OS-level CPU/memory/tool dynamics | 当前 OS-level resource characterization 核心工作。 |
| 2026 | arXiv | Cloud-OpsBench | https://arxiv.org/abs/2603.00468 | Kubernetes RCA，452 fault cases，40 root cause types，state snapshot paradigm | 包含 state snapshot, alerts/logs/tool cache | cloud-native agent workload，适合 AKernel ops agent evaluation。 |
| 2026 | arXiv | CodeTracer / CodeTraceBench | https://arxiv.org/abs/2604.11641 | coding-agent state tracing/failure localization | structured hierarchical trace, step-level annotations | AKernel observability/provenance 层可借鉴。 |
| 2026 | arXiv | SWE-chat | https://arxiv.org/abs/2604.20779 | 6000 real coding-agent sessions，63K prompts，355K tool calls | 真实 session/tool trace；非 OS metrics | 真实用户场景最强证据之一，可补 AgentCgroup benchmark-only 限制。 |
| 2026 | arXiv | Agentic AI Workload Characteristics | https://arxiv.org/abs/2605.26297 | ReAct agents；LLM serving + tool execution end-to-end tracing | 有 token/tool/KV-cache tracing，偏 serving | 支撑 long-lived KV cache、decode-dominated、tool behavior temporal structure。 |
| 2026 | arXiv | TraceLab | https://arxiv.org/abs/2606.30560 | 4300 coding-agent sessions，350K LLM steps，430K tool calls，Claude Code/Codex | 真实 serving/tool trace；不是完整 OS resource trace | AKernel workload model 的核心公开 trace。 |
| 2026 | arXiv | Microservice Failure Diagnosis Benchmark / AgentOpsEval | https://arxiv.org/abs/2606.29193 | 500+ expert-labeled failure cases，OpenTelemetry Demo Store/HipsterShop | 多模态 observability data | cloud ops agent benchmark，可用于 AKernel Agent Realm/Policy/Audit 评测。 |

### Trace 工作的共同缺口

目前公开 trace 的缺口非常明显：

- BurstGPT 有 LLM serving trace，但没有 tool/sandbox OS metrics。
- SWE-bench/SWE-rebench 有 executable tasks，但默认没有 OS resource trace。
- TraceLab/SWE-chat 有真实 coding-agent sessions/tool calls，但不是完整 cgroup/filesystem/network/object locality trace。
- AIOpsLab/Cloud-OpsBench 有 cloud telemetry，但不是大量 sandboxed agent OS resource trace。
- AgentCgroup 有 OS-level resource trace，但规模较小，且主要是 coding-agent benchmark task。

AKernel 最好的贡献机会是：**把这些 trace 维度合起来**，形成 realm/session/tool/process/object/policy/resource 的 production trace。

## 5. AI/ML Benchmarks：Agent Workload 来源库

### 5.1 Coding / Terminal / ML Engineering

| 年份 | 会议/来源 | 工作 | URL | 任务类型 | Trace / resource | 关系 |
|---|---|---|---|---|---|---|
| 2024 | ICLR | SWE-bench | https://proceedings.iclr.cc/paper_files/paper/2024/hash/edac78c3e300629acfe6cbe9ca88fb84-Abstract-Conference.html | 真实 GitHub issue 修复，2294 tasks | 真实 issue/PR；无 OS resource trace | coding-agent 基础 benchmark。 |
| 2024 | NeurIPS | SWE-agent | https://proceedings.neurips.cc/paper_files/paper/2024/file/5a7c947568c1b1328ccc5230172e1e7c-Paper-Conference.pdf | agent-computer interface for SWE | 可产生 command/action logs | 命令/API 面可映射到 AgentCgroup tool boundary。 |
| 2024 | ICML | R2E | https://proceedings.mlr.press/v235/jain24c.html | GitHub repo to executable programming environment | 真实代码仓库；无 resource trace | 自动构建 sandbox/test env。 |
| 2024 | OpenAI/SWE-bench | SWE-bench Verified | https://www.swebench.com/verified.html | 500 人工验证 solvable issue tasks | 真实 issue；无 OS trace | 更可靠 regression benchmark。 |
| 2025 | ICLR | MLE-bench | https://proceedings.iclr.cc/paper_files/paper/2025/hash/7e3767db483c942b883eb4f8cfb74e31-Abstract-Conference.html | Kaggle ML engineering | 真实 competitions/leaderboard；无统一 OS trace | GPU/CPU/storage/resource budget 很适合 AKernel。 |
| 2025 | ICML | SWE-Gym | https://proceedings.mlr.press/v267/pan25g.html | SWE agent training env | 真实 GitHub tasks；sampled trajectories | 训练侧 sandbox/verifier workload。 |
| 2025 | ICLR | SWE-bench Multimodal | https://proceedings.iclr.cc/paper_files/paper/2025/hash/07d6332ae36730707fddddba736d7b6c-Abstract-Conference.html | visual frontend/JS bug fixing | screenshot + code/test | GUI/code/test mixed resource workload。 |
| 2025 | NeurIPS D&B | SWE-rebench | https://arxiv.org/abs/2505.20411 | 21k+ interactive Python SWE tasks | executable tasks | AgentCgroup 使用；AKernel coding workload source。 |
| 2025 | NeurIPS D&B | SWE-bench-Live | https://arxiv.org/abs/2505.23419 | live-updatable SWE benchmark | Docker images | 持续刷新环境，适合 AKernel image/object cache evaluation。 |
| 2026 | arXiv | Terminal-Bench 2.0 | https://arxiv.org/abs/2601.11868 | hard terminal tasks | command/process trace 可采 | CLI sandbox/process-tree 控制核心 workload。 |
| 2026 | arXiv | SWE-chat | https://arxiv.org/abs/2604.20779 | real coding-agent sessions | 6000 sessions, 355K tool calls | 补足 curated benchmark 与真实使用差异。 |
| 2026 | arXiv | TraceLab | https://arxiv.org/abs/2606.30560 | real coding-agent serving workload | 4300 sessions, 350K LLM steps, 430K tool calls | 真实 session/tool/KV/cache workload。 |

### 5.2 Web / GUI / Computer Use / Mobile

| 年份 | 会议/来源 | 工作 | URL | 任务类型 | Trace / resource | 关系 |
|---|---|---|---|---|---|---|
| 2023 | NeurIPS D&B | Mind2Web | https://arxiv.org/abs/2306.06070 | 真实网站导航 | 众包 action sequences；无 resource trace | 浏览器 tool-call 模型。 |
| 2024 | ICLR | AgentBench | https://proceedings.iclr.cc/paper_files/paper/2024/hash/e9df36b21ff4ee211a8b71ee8b7e9f57-Abstract-Conference.html | OS/DB/Web/game 等 8 类交互环境 | 交互日志潜力；无 OS trace | 基础 agent task suite。 |
| 2024 | ICLR | WebArena | https://proceedings.iclr.cc/paper_files/paper/2024/hash/4410c0711e9154a7a2d26f9b3816d1ef-Abstract-Conference.html | self-hosted realistic websites | environment state；无 resource trace | 浏览器沙箱/网络/状态 reset。 |
| 2024 | ICLR | GAIA | https://proceedings.iclr.cc/paper_files/paper/2024/hash/25ae35b5b1738d80f1f03a8713e405ec-Abstract-Conference.html | general assistant tasks | 无 resource trace | tool permission/network access policy。 |
| 2024 | ACL | VisualWebArena | https://aclanthology.org/2024.acl-long.50/ | multimodal web tasks | screenshot/action logs | 浏览器截图/DOM/action objects。 |
| 2024 | ICML | WebLINX | https://arxiv.org/abs/2402.05930 | conversational web navigation | 100K interactions, 2300 expert demos | 真实浏览器 action/observation trace schema。 |
| 2024 | ICML | WorkArena | https://arxiv.org/abs/2403.07718 | ServiceNow enterprise workflow | enterprise-style tasks | 企业 SaaS state/permission/audit。 |
| 2024 | NeurIPS | OSWorld | https://papers.nips.cc/paper_files/paper/2024/hash/5d413e48f84dc61244b6be550f1cd8f5-Abstract-Datasets_and_Benchmarks_Track.html | real OS/desktop app tasks | environment interaction logs 可采 | desktop sandbox、process isolation、rollback。 |
| 2024 | NeurIPS | WorkArena++ | https://arxiv.org/abs/2407.05291 | compositional enterprise workflows | no resource trace | 长流程 checkpoint/state recovery。 |
| 2024 | ACL | AppWorld | https://aclanthology.org/2024.acl-long.850/ | multi-app API + code interaction | simulated user digital activity | API permission, cross-app state consistency。 |
| 2024 | EMNLP | AssistantBench | https://aclanthology.org/2024.emnlp-main.505.pdf | time-consuming open web tasks | real website tasks | long-running network access, timeout/retry policy。 |
| 2025 | ICLR | AndroidWorld | https://proceedings.iclr.cc/paper_files/paper/2025/file/01a83bc2f2732a58e6aa731e659e7101-Paper-Conference.pdf | Android app control | dynamic tasks | mobile GUI sandbox/app permission/state reset。 |
| 2025 | TMLR | BrowserGym | https://www.servicenow.com/research/publication/thibault-le-sellier-de-chezelles-the-tmlr2025.html | Web agent Gym ecosystem | wraps WebArena/WorkArena/MiniWoB | unified observation/action API。 |
| 2025 | arXiv | Online-Mind2Web | https://arxiv.org/abs/2504.01382 | online real web tasks | dynamic real web | 需要 record/replay/network snapshot。 |
| 2025 | arXiv/OpenReview | ComputerRL | https://arxiv.org/abs/2508.14040 | desktop agent RL, thousands of virtual desktops | large parallel rollout；resource trace 可采 | 大规模 computer-use sandbox pool，AKernel 高相关。 |

### 5.3 Tool / API / Customer-Service Agents

| 年份 | 会议/来源 | 工作 | URL | 任务类型 | Trace / resource | 关系 |
|---|---|---|---|---|---|---|
| 2023 | EMNLP | API-Bank | https://arxiv.org/abs/2304.08244 | 73 API tools，多轮 API planning/execution | API logs | tool gateway/policy baseline。 |
| 2024 | ICLR | ToolLLM / ToolBench | https://iclr.cc/virtual/2024/poster/18267 | 大规模 API/tool use | API call trajectories | API rate limit, auditing, capability。 |
| 2025 | ICLR | tau-bench | https://iclr.cc/virtual/2025/poster/28170 | tool-agent-user interaction, airline/retail | historical trajectories dir；simulated users | pass^k reliability, stateful DB/tool environment。 |
| 2025/2026 | Sierra / arXiv | tau2/tau3/tau-Voice | https://arxiv.org/abs/2506.07982 | customer-service, voice, dual-control | dialogue/tool/db/audio trace | long session, multimodal stream, transactional state。 |

### 5.4 Multi-Agent / Enterprise / Cloud Ops

| 年份 | 会议/来源 | 工作 | URL | 任务类型 | Trace / resource | 关系 |
|---|---|---|---|---|---|---|
| 2023 | NeurIPS | CAMEL | https://arxiv.org/abs/2303.17760 | role-playing communicative agents | conversation trace | multi-agent role/state isolation。 |
| 2023/2024 | ACL | ChatDev | https://arxiv.org/abs/2307.07924 | virtual software company | dialogue/code artifacts | process graph/artifact handoff。 |
| 2023 | Microsoft | AutoGen | https://arxiv.org/abs/2308.08155 | programmable multi-agent conversation | conversation/tool logs | multi-agent runtime pattern。 |
| 2023/2024 | ICLR | MetaGPT | https://arxiv.org/abs/2308.00352 | SOP multi-agent software development | artifact logs | workflow-as-code, role isolation。 |
| 2024 | ICLR | SOTOPIA | https://proceedings.iclr.cc/paper_files/paper/2024/hash/b3075b88e583a0e98d8b24338a613060-Abstract-Conference.html | social interaction/negotiation | generated interaction | multi-agent session isolation。 |
| 2025 | MLSys | AIOpsLab | https://arxiv.org/abs/2501.06706 | cloud ops, fault injection, telemetry | telemetry export | AKernel AgentOps benchmark。 |
| 2025 | ICML | ITBench | https://arxiv.org/abs/2502.05352 | SRE/CISO/FinOps IT automation | partial ops trace | enterprise IT permissions/audit/compliance。 |
| 2025 | ACL | MultiAgentBench | https://aclanthology.org/2025.acl-long.421/ | collaboration/competition topologies | no resource trace | group/realm scheduling, shared memory topology。 |
| 2025 | NeurIPS D&B | TheAgentCompany | https://proceedings.neurips.cc/paper_files/paper/2025/hash/0d744742f6fac4d1134c019b7cef3c8a-Abstract-Datasets_and_Benchmarks_Track.html | simulated company, web/code/program/colleague communication | rich interaction trace possible | 最像 agent cloud OS application：identity, workflow, collaboration, permissions。 |
| 2026 | arXiv | Cloud-OpsBench | https://arxiv.org/abs/2603.00468 | Kubernetes RCA | state snapshot, alerts/logs/tool cache | Cloud-native AKernel eval candidate。 |
| 2026 | arXiv | Microservice Failure Diagnosis Benchmark | https://arxiv.org/abs/2606.29193 | microservice AgentOps failure diagnosis | multimodal observability data | failure diagnosis trace/evidence/provenance。 |

## 6. Industry / Blog / Product Sources

这些资料不应当作为论文核心 related work，但很适合作为 motivation 和系统需求证据。尤其是 workload shape、telemetry、pricing、sandbox APIs 和 production constraints。

### 6.1 Coding-agent / agent loop / tracing

| 年份 | 来源 | URL | 内容 | Metrics / workload | 关系 |
|---|---|---|---|---|---|
| 2024 | LangGraph | https://www.langchain.com/blog/langgraph | 循环图 agent runtime：LLM 决策、tool execution、再回 LLM | 框架模式，无 OS metrics | 说明 agent 是循环型、状态型执行。 |
| 2024 | LangGraph interrupt/HITL | https://www.langchain.com/blog/making-it-easier-to-build-human-in-the-loop-agents-with-interrupt | persistence/checkpoint 作为一等能力，可暂停、人工审核、修改 checkpoint 后恢复 | workflow checkpoint semantics | 与 AKernel Agent Process checkpoint/HITL 对齐。 |
| 2025 | OpenAI Codex | https://openai.com/index/introducing-codex/ | 云端 software engineering agent，任务通常 1-30 分钟，提供 terminal logs/test outputs | task duration, terminal/test evidence | 支持 coding agent 是分钟级长任务。 |
| 2026 | OpenAI Codex agent loop | https://openai.com/index/unrolling-the-codex-agent-loop/ | Codex CLI harness/agent loop：user input, model reasoning, tool calls, context construction | loop structure | AKernel 可把 agent loop 作为 OS scheduling object。 |
| 2026 | OpenAI WebSocket latency | https://openai.com/index/speeding-up-agentic-workflows-with-websockets/ | latency 来自 API service, model inference, client-side tools/context；WebSocket/connection cache 优化 overhead | latency breakdown categories | AgentCgroup/AKernel 可拆分 tool runtime/context construction。 |
| 2026 | OpenAI Agents SDK tracing | https://developers.openai.com/api/docs/guides/agents/integrations-observability | 默认 tracing：model calls, tool calls, handoffs, guardrails, custom spans | standardized spans/evals | AKernel 可把 OS/cgroup metrics 与 agent spans 对齐。 |
| 2025 | Agent Lightning | https://www.microsoft.com/en-us/research/project/agent-lightning/ | agent execution 与 RL training 解耦，每一步 agent 行为转成 RL data | trajectories/traces | runtime trace 可成为训练数据，不只是 debug。 |
| 2025 | Agent Lightning traces | https://microsoft.github.io/agent-lightning/latest/tutorials/traces/ | trace spans/reward spans 作为 first-class primitive | span data | 可将 resource events 与 reward spans 对齐。 |
| 2025 | LangSmith dashboards | https://docs.langchain.com/langsmith/dashboards | trace count, latency, error rate, token usage, cost, tool latency/error | LLM/tool metrics | AKernel observability baseline；需要补 OS metrics。 |

### 6.2 Sandboxing / containment / execution platforms

| 年份 | 来源 | URL | 内容 | Metrics / workload | 关系 |
|---|---|---|---|---|---|
| 2024 | AutoGen Docker default execution | https://microsoft.github.io/autogen/0.2/blog/2024/01/23/Code-execution-in-docker/ | AutoGen 默认把 code execution 放入 Docker | security policy | 支持 agent code execution 默认隔离。 |
| 2025 | Anthropic Claude Code sandboxing | https://www.anthropic.com/engineering/claude-code-sandboxing | 文件系统和网络隔离；permission prompts 减少 84% | 84% prompt reduction | 说明 containment 能减少人工审批疲劳。 |
| 2026 | Anthropic Managed Agents | https://www.anthropic.com/engineering/managed-agents | session/harness/sandbox 三个虚拟化接口；session 是 append-only log | session log model | 与 AKernel control/execution/log plane 映射很强。 |
| 2026 | Anthropic containment | https://www.anthropic.com/engineering/how-we-contain-claude | 风险 = failure probability + blast radius；Claude Code 用户约 93% 会批准 permission prompts | 93% approval telemetry | 支撑边界优于频繁人工 prompt approval。 |
| 2026 | E2B sandbox metrics | https://e2b.dev/docs/sandbox/metrics | CPU/memory/disk usage metrics，每 5 秒采集；按 sandbox compute resources per-second billing | CPU/mem/disk metrics | AgentCgroup 可把指标细化到 per-tool/per-agent。 |
| 2026 | Modal sandboxes | https://modal.com/docs/guide/sandboxes | secure containers for untrusted user/agent code；GPU, per-second pricing, warm pool | platform metrics/examples | sandbox commodity baseline。 |
| 2026 | Daytona telemetry / RL infra | https://www.daytona.io/docs/en/observability/otel-collection/ | Logs/Traces/Metrics，RL infra bench 测 Docker/EC2/K8s/ECS/Daytona startup/per-action latency | latency, worker-hours | 很适合 AKernel RL rollout/sandbox infra comparison。 |
| 2026 | Cloudflare Sandboxes GA | https://blog.cloudflare.com/sandbox-ga/ | agent coding workload：clone repo, build, dev server, credential injection, PTY, persistent interpreters, background processes, preview URLs, snapshots, Active CPU Pricing | workload pattern/cost model | Active CPU/cost accounting 对 AKernel 有参考价值。 |
| 2026 | Cloudflare Agents runtime | https://developers.cloudflare.com/agents/ | durable identity, SQL, realtime, scheduled work, recoverable execution, logs/metrics/traces | logs/metrics/traces, scale target | agent session 需要 durable state/recoverable execution。 |
| 2025/2026 | GKE Agent Sandbox | https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox | isolated/stateful/singleton agent workloads, warm pools, pod snapshots, gVisor/Kata | architecture/cold-start direction | 说明 Kubernetes 在原语化 agent sandbox；AKernel 要更细粒度。 |
| 2026 | Azure Foundry Agents Code Interpreter | https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/code-interpreter | sandboxed Python/shell code interpreter, custom interpreter via ACA Dynamic Sessions | sandbox capability, pricing | 跨云统一 resource/cost/trace interface 是 AKernel 空间。 |
| 2026 | Docker Sandboxes / MCP Toolkit | https://docs.docker.com/ai/sandboxes/ | 管理 AI coding agent 的 network/filesystem policies，MCP tools containerization | security/tool runtime | enterprise local/dev environment baseline。 |
| 2026 | OpenSandbox | https://github.com/opensandbox-group/OpenSandbox | 多语言 SDK、Docker/K8s runtime，coding/GUI/eval/RL workload | workload types | sandbox API baseline；AKernel 补 resource/provenance/object。 |
| 2026 | Cube Sandbox | https://cubesandbox.com/blog/posts/2026-06-03-cubesandbox-v0.3.0-snapshot.html | snapshot creation/from-snapshot/rollback/clone benchmark，面向 RL/high-concurrency workloads | snapshot benchmark | 对 AKernel C/R/fork baseline 很重要。 |
| 2026 | Vercel Sandbox | https://vercel.com/blog/vercel-sandbox-is-now-generally-available | Firecracker microVM sandbox for untrusted code | platform capability | industry proof that sandbox is commodity; AKernel must go beyond sandbox。 |

### 6.3 Benchmark sites / leaderboards

| 年份 | 来源 | URL | 内容 | Metrics / workload | 关系 |
|---|---|---|---|---|---|
| 2023-2026 | SWE-bench | https://www.swebench.com/ | original/verified/multimodal/live leaderboards, resolved%, avg cost | resolved%, cost, trajectories | cost-to-solution 和 success rate 可作为 AKernel goodput 指标。 |
| 2025/2026 | Terminal-Bench / Harbor | https://www.tbench.ai/news/announcement-2-0 | cloud-scale terminal-agent benchmark, Harbor supports thousands of containers and RL/SFT rollouts | terminal tasks, rollout infra | AKernel terminal/RL rollout benchmark。 |
| 2026 | Terminal-Bench site | https://www.tbench.ai/ | realistic terminal tasks：Linux kernel, git/web server, archive 等 | task examples/leaderboard | 长命令、后台服务、网络端口、文件状态管理。 |

## 7. 与 AgentCgroup 的技术轴线

可以把相关工作放到下面几个轴上：

| 技术轴 | 代表工作 | 共同问题 | AKernel 可做的扩展 |
|---|---|---|---|
| Resource control | AgentCgroup, CPU-centric agentic execution, Cortex | agent resource demand 随 tool/stage 快速变化 | 从 per-container quota 扩展到 per-realm/per-tool/per-object accounting |
| C/R and exploration | Fork-Explore-Commit, Crab, DeltaBox, ACRFence, Cube snapshot | agent 需要 branch/rollback/retry，但外部 side effects 不一定可逆 | 把 checkpoint、object diff、tool effect、policy log 合并成 state plane |
| Security and containment | Sandlock, Grimlock, Anthropic containment, Docker/GKE/OpenSandbox | agent 会执行 untrusted code/tool/API | 将 capability/resource/provenance 绑定到 Agent Process |
| LLM serving | Parrot, AgentServe, Continuum, Helium, Agentix, Kairos | agent 多轮 LLM 调用与 tool gaps 造成 KV/cache/scheduling 问题 | 联合 model-side context/KV 和 sandbox-side process/object locality |
| Workflow orchestration | Murakkab, Compound AI Systems, LangGraph, Agent Lightning | orchestration 与 resource management 脱节 | Datacenter Function / Agent Realm 作为可优化 workflow unit |
| Workload traces | TraceLab, SWE-chat, Agentic AI Workload Characteristics, AIOpsLab | benchmark/trace 往往缺 OS resource/object/policy 维度 | 生产 trace：session + process + tool + object + policy + resource |
| Benchmarks | SWE-bench, Terminal-Bench, OSWorld, WebArena, WorkArena, MLE-bench | benchmark 关注 success，少关注 resource/cost/locality | 加 resource/cost/goodput wrapper，形成 AKernel evaluation suite |

## 8. 对 AKernel 的直接启示

### 8.1 论文 motivation 可以这么写

公开资料显示 agent workload 已经被多条独立路线确认：

- OS 侧：AgentCgroup/Crab/DeltaBox/Sandlock/Fork-Explore-Commit 证明 agent 需要新的 OS primitives。
- Serving 侧：AgentServe/Helium/Continuum/Murakkab/Parrot 证明 agent workflow 不是独立 LLM request。
- Benchmark 侧：SWE-bench/Terminal-Bench/OSWorld/AIOpsLab 证明 agent 需要真实 execution environment。
- Industry 侧：OpenAI/Anthropic/E2B/Cloudflare/GKE/Daytona/Cube 都在把 sandbox、trace、checkpoint、cost、containment 产品化。

AKernel 应该把这些观察统一成一个 stronger claim：

> Agent workload 的基本系统单元不是 request、container、pod、function，也不是单个 sandbox，而是一个 stateful, policy-carrying, artifact-producing Agent Process。多个 Agent Process 组成 Agent Realm，并通过 Agent Object 共享、恢复、迁移、审计和调度。

### 8.2 论文最该补的 trace

建议 AKernel 采集比 AgentCgroup 更完整的 trace：

| 层级 | 字段 | 为什么比 AgentCgroup 更强 |
|---|---|---|
| Realm/session | duration, status, success, retry, human gap, cost, SLO | 从 single task resource timeline 升级到 agent-goodput |
| Process/sandbox | create, start, fork, checkpoint, restore, hibernate, migrate, commit, fail | 证明 agent lifecycle 是 OS primitive |
| Tool/call | shell, file, test, browser, network, API, DB, model call, duration, resource usage | 对齐 AgentCgroup 的 tool-call boundary |
| Object/artifact | repo, diff, logs, checkpoint, cache, dataset, browser trace, size, reuse distance, location | 证明 object locality 是 datacenter issue |
| Policy/provenance | credential, egress, blocked action, irreversible effect, artifact refs | 回应 ACRFence/AgentTrust/provenance 方向 |
| Model/context | prompt length, output length, KV/cache hit, tool gap, prefill/decode | 对齐 AgentServe/Continuum/TraceLab |

### 8.3 推荐 evaluation benchmark 组合

| Workload | 来源 | 目的 |
|---|---|---|
| Coding agent | SWE-bench Verified, SWE-rebench, SWE-bench-Live, TraceLab/SWE-chat style internal trace | coding agent resource and object characterization |
| Terminal agent | Terminal-Bench 2.0 | shell/process tree/file/network control |
| Computer-use | OSWorld, BrowserGym/WebArena/WorkArena | GUI/browser state and replay |
| Cloud ops | AIOpsLab, Cloud-OpsBench, Microservice Failure Diagnosis | AgentOps / Datacenter Use |
| Agentic RL rollout | SWE-Gym, ComputerRL, Terminal-Bench Harbor | burst, fork, checkpoint, rollback, environment pool |
| Multi-agent | MultiAgentBench, TheAgentCompany, AutoGen/MetaGPT workflows | Agent Realm / shared object / group policy |
| ML/data | MLE-bench | CPU/GPU/storage/data locality |

### 8.4 AKernel 相对 AgentCgroup 的差异点

AgentCgroup 可以作为 AKernel 的底层机制之一，但 AKernel 论文要避免只做 “AgentCgroup at cluster scale”。更强的差异点是：

1. **从 tool-call resource control 到 realm-level lifecycle management**：AgentCgroup 关心单 agent tool-call 的 CPU/memory；AKernel 关心 Agent Realm 的启动、分支、恢复、迁移、提交、失败和成功率。
2. **从 cgroup hierarchy 到 Agent Object locality**：AgentCgroup 不管理 artifact/cache/checkpoint/object placement；AKernel 可以证明 object locality 对成本和 latency 关键。
3. **从 resource isolation 到 policy/provenance-carrying execution**：AgentCgroup 主要是 resource；AKernel 需要把 credential/network/irreversible side effect/audit 作为 OS state。
4. **从 benchmark task 到 production trace**：AgentCgroup 使用 144 SWE-rebench tasks；AKernel 应采集真实 sessions，覆盖 coding、terminal、web、cloud ops、RL rollout、多 agent。
5. **从 single host/sandbox to datacenter agent OS**：AgentCgroup 可以控制单 host 或 multi-tenant sandbox；AKernel 要证明跨节点 placement、cache、C/R、policy、deployment 的端到端收益。

## 9. 建议论文相关工作写法

### 9.1 不要按“AI agent 论文”平铺

建议 Related Work 按系统问题组织：

1. **Agent workload characterization**：AgentCgroup, TraceLab, SWE-chat, Agentic AI Workload Characteristics, AIOpsLab, BurstGPT。
2. **Agent OS primitives**：AIOS, Fork-Explore-Commit, Crab, DeltaBox, Sandlock, ACRFence, Rethinking OS Interfaces, AgenticOS workshop。
3. **Agentic serving and workflow scheduling**：Parrot, AgentServe, Helium, Murakkab, Cortex, Continuum, Kairos, Agentix, Pie。
4. **Agent benchmarks and environments**：SWE-bench series, Terminal-Bench, OSWorld, WebArena, WorkArena, MLE-bench, BrowserGym, AIOpsLab, Cloud-OpsBench。
5. **Industry sandbox/observability platforms**：OpenAI Codex/Agents tracing, Anthropic Managed Agents/containment, E2B metrics, Cloudflare Sandboxes, GKE Agent Sandbox, Cube, Daytona, Modal, OpenSandbox。

### 9.2 可直接使用的 contrast 句

> AgentCgroup shows that OS resource demand in sandboxed coding agents is tool-call driven and poorly matched by container-level policies. AKernel generalizes this observation from resource control to a cloud operating system abstraction: Agent Processes are stateful, policy-carrying execution units inside Agent Realms, and their performance depends jointly on tool-call resources, sandbox state, object locality, checkpoint/restore, model context, and provenance.

> Recent agent serving systems optimize LLM-side execution across multi-turn workflows, while AgentCgroup and OS-for-agent systems optimize sandbox-side execution. AKernel targets the missing cross-layer boundary: scheduling and managing agent execution across model calls, tools, objects, sandboxes, and datacenter nodes.

## 10. 优先引用清单

如果论文篇幅有限，建议优先读/引用这些：

### 必引：Agent OS/resource/state

1. AgentCgroup：https://arxiv.org/abs/2602.09345
2. Crab：https://arxiv.org/abs/2604.28138
3. Fork, Explore, Commit：https://arxiv.org/abs/2602.08199
4. DeltaBox：https://arxiv.org/abs/2605.22781
5. Sandlock：https://arxiv.org/abs/2605.26298
6. ACRFence：https://arxiv.org/abs/2603.20625
7. Rethinking OS Interfaces / DMI：https://arxiv.org/abs/2510.04607
8. AIOS：https://arxiv.org/abs/2403.16971
9. AgenticOS workshop：https://os-for-agent.github.io/asplos-2026.html

### 必引：agent serving/workflow

1. Parrot：https://www.usenix.org/conference/osdi24/presentation/lin-chaofan
2. AgentServe：https://arxiv.org/abs/2603.10342
3. Helium：https://arxiv.org/abs/2603.16104
4. Murakkab：https://arxiv.org/abs/2508.18298
5. Continuum：https://arxiv.org/abs/2511.02230
6. Kairos：https://arxiv.org/abs/2508.06948
7. Cortex：https://arxiv.org/abs/2510.14126
8. CPU-Centric Agentic Execution：https://arxiv.org/abs/2511.00739

### 必引：workload/trace

1. TraceLab：https://arxiv.org/abs/2606.30560
2. SWE-chat：https://arxiv.org/abs/2604.20779
3. Agentic AI Workload Characteristics：https://arxiv.org/abs/2605.26297
4. AIOpsLab：https://arxiv.org/abs/2501.06706
5. Cloud-OpsBench：https://arxiv.org/abs/2603.00468
6. Terminal-Bench 2.0：https://arxiv.org/abs/2601.11868
7. SWE-rebench：https://arxiv.org/abs/2505.20411
8. BurstGPT：https://arxiv.org/abs/2401.17644

### 必引：benchmark sources

1. SWE-bench：https://www.swebench.com/
2. OSWorld：https://arxiv.org/abs/2404.07972
3. WebArena：https://proceedings.iclr.cc/paper_files/paper/2024/hash/4410c0711e9154a7a2d26f9b3816d1ef-Abstract-Conference.html
4. WorkArena：https://arxiv.org/abs/2403.07718
5. MLE-bench：https://proceedings.iclr.cc/paper_files/paper/2025/hash/7e3767db483c942b883eb4f8cfb74e31-Abstract-Conference.html
6. BrowserGym：https://www.servicenow.com/research/publication/thibault-le-sellier-de-chezelles-the-tmlr2025.html
7. MultiAgentBench：https://aclanthology.org/2025.acl-long.421/
8. TheAgentCompany：https://proceedings.neurips.cc/paper_files/paper/2025/hash/0d744742f6fac4d1134c019b7cef3c8a-Abstract-Datasets_and_Benchmarks_Track.html

### 工业资料：motivation/feature comparison

1. E2B metrics：https://e2b.dev/docs/sandbox/metrics
2. Daytona telemetry：https://www.daytona.io/docs/en/observability/otel-collection/
3. Cloudflare Sandboxes：https://blog.cloudflare.com/sandbox-ga/
4. GKE Agent Sandbox：https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox
5. Anthropic containment：https://www.anthropic.com/engineering/how-we-contain-claude
6. Anthropic Managed Agents：https://www.anthropic.com/engineering/managed-agents
7. OpenAI Agents tracing：https://developers.openai.com/api/docs/guides/agents/integrations-observability
8. Cube Sandbox snapshot benchmark：https://cubesandbox.com/blog/posts/2026-06-03-cubesandbox-v0.3.0-snapshot.html

## 11. 结论：现在 agent workload 论文主要有哪些？

如果按“最主要、最相关”排序：

1. **AgentCgroup**：OS-level resource dynamics/control 的核心论文。
2. **TraceLab / SWE-chat**：真实 coding-agent workload trace 的核心资料。
3. **Agentic AI Workload Characteristics**：LLM serving + tool execution 的 end-to-end characterization。
4. **Crab / Fork-Explore-Commit / DeltaBox / Sandlock / ACRFence**：agent sandbox state/security/rollback 的 OS primitives。
5. **Murakkab / Helium / AgentServe / Continuum / Kairos / Parrot**：agentic workflow serving/resource scheduling。
6. **SWE-bench/SWE-rebench/SWE-bench-Live/Terminal-Bench/OSWorld/AIOpsLab/Cloud-OpsBench**：主要 benchmark/workload sources。
7. **OpenAI/Anthropic/E2B/Cloudflare/GKE/Daytona/Cube 等 blogs/docs**：证明产业侧已经把 agent sandbox、trace、cost、checkpoint、observability 产品化。

对 AKernel 来说，最重要的 research gap 是：

> 现有工作要么研究单 agent/sandbox 的 OS resource control，要么研究 LLM-side agent serving，要么提供 agent benchmark/trace，要么提供 sandbox 产品能力。还缺少一个把 agent session、tool-call resources、sandbox state、object artifacts、policy/provenance、checkpoint/rollback、multi-node placement 和 cost-to-solution 统一起来的 cloud OS。

这正是 AKernel 可以占住的位置。

