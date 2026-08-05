# Agent Sandbox、AX 与 Agent Substrate：功能边界、源码架构与相互关系调研

> 调研时间：2026-08-05 10:56:37（Asia/Shanghai）
>
> 文档时间戳：`20260805T025637Z`
>
> Agent Sandbox commit：`ac67d375a8b00c305a8647cdfa4aa876f43521e7`（2026-08-04）
>
> AX commit：`f327e23b5b842e9b700675ded9a6cdb79c505856`（2026-07-28）
>
> Agent Substrate commit：`cbdeb7dbe003a55a16960a301bc595d9aa38b1ad`（2026-07-31）
>
> AX 实际固定的 Substrate commit：`3cb7433bd8a81a1f9aa54ef98418fbde8205fc42`（2026-07-06）
>
> 方法：固定版本源码、CRD、controller、SDK、部署清单和项目文档交叉核对；当前未在真实 GKE/Kubernetes 集群部署，也未复跑性能测试。

## 1. 结论摘要

### 1.1 三者不是同一层的三个同类产品

最准确的分层是：

~~~text
AX
  = Agent application / harness execution layer
  = conversation、execution turn、event log、single-writer边界、输出流

Agent Substrate
  = high-frequency logical Actor control plane
  = Actor状态、Worker分配、snapshot suspend/resume、请求触发唤醒和路由

Agent Sandbox
  = Kubernetes-native Sandbox API and controller
  = Sandbox CRD、Pod/PVC/Service编排、SandboxTemplate/Claim/WarmPool
~~~

所以不能简单画成：

~~~text
Agent Sandbox -> AX -> Substrate
~~~

当前源码真正成立的依赖只有：

~~~text
AX --已实现Substrate Harness adapter--> Agent Substrate
~~~

`agent-sandbox` 与另外两个仓库当前没有 import、API adapter、CRD 引用或联合部署清单。它和 Substrate 是解决相近问题的两种 runtime/control-plane 架构，既不是 Substrate 的下层 runtime，也不是 AX 当前已经支持的 compute backend。

### 1.2 “GKE Agent Sandbox”与开源 Agent Sandbox 的关系

用户提到的 GitHub 仓库 [`kubernetes-sigs/agent-sandbox`](https://github.com/kubernetes-sigs/agent-sandbox) 是 Kubernetes SIG Apps 下的开源项目：核心是 `Sandbox` CRD/controller，原则上可安装到任意满足依赖的 Kubernetes 集群。项目自己把它定义为 **sandbox orchestrator**，底层隔离委托给由 `RuntimeClass` 选择的 gVisor、Kata 等 runtime，而不是在这个仓库里重新实现 VMM或容器内核。[`README.md:15-20`](../../agent-substrate/agent-sandbox/README.md#L15)

**GKE Agent Sandbox add-on** 是 Google 对同一个项目的托管安装形态：Google 托管 controller/CRD，并接入 GKE 的 gVisor node、admission policy 等。当前本地文档还指出 API 版本差异：GKE add-on 为 `v1alpha1`，本地 OSS HEAD 主要使用 `v1beta1`。[`anthropic-managed-agents/_index.md:192-215`](../../agent-substrate/agent-sandbox/site/content/docs/use-cases/anthropic-managed-agents/_index.md#L192)

因此本文使用以下术语：

- **Agent Sandbox**：`kubernetes-sigs/agent-sandbox` 开源上游；
- **GKE Agent Sandbox add-on**：该上游在 GKE 中的 Google-managed 安装和云能力组合；
- **GKE PodSnapshot**：Agent Sandbox Python 扩展依赖的 GKE 专有 snapshot CRD/controller，不等于 OSS core controller 自己实现了通用内存快照。

### 1.3 Agent Sandbox 和 Substrate 最本质的差别

两者都使用 Kubernetes、Pod、gVisor/Kata、template 和 warm capacity，但对象粒度完全不同：

| 问题 | Agent Sandbox | Agent Substrate |
|---|---|---|
| 用户逻辑实例 | `Sandbox` / `SandboxClaim` Kubernetes CR | Redis/ValKey 中的 logical `Actor` record |
| 活跃执行载体 | 每个 Sandbox 一个 backing Pod | Actor restore 到某个预热 Worker Pod 内部 |
| Kubernetes动态对象规模 | 大致随 Sandbox/Claim/Pod 数增长 | Worker Pod 随 active capacity 增长；Actor 状态不进 etcd |
| warm pool | 预创建完整 Sandbox CR + Pod | 预创建较少的外层 Worker Pod slot |
| 热分配 | Claim 通过 Kubernetes controller adopt 一个 Sandbox | ateapi 从内存/Redis Worker 集合 claim 空闲 assignment |
| suspend | OSS core 删除 Pod、保留 Sandbox/PVC | checkpoint sandbox，记录 snapshot，释放 Worker |
| resume | Kubernetes controller 重新创建/恢复 Pod | direct RPC 到 atelet/ateom，在现有 Worker 内 restore |
| per-instance K8s scheduler | cold create/resume 会走；warm claim已提前走过 | per-Actor resume 不走 kube-scheduler |
| 同一外层 Pod多实例 | 当前没有，roadmap 才计划 | 同时最多一个 Actor；时间上可依次承载很多 Actor |

若用 `S` 表示 Agent Sandbox 实例数，`A` 表示 Substrate Actor 数，`W` 表示 Substrate Worker 数，则：

~~~text
Agent Sandbox:
  Kubernetes Sandbox CR / backing Pod ≈ O(S)

Agent Substrate:
  Kubernetes Worker Pod ≈ O(W)
  Redis logical Actor ≈ O(A)
  设计目标是 W << A
~~~

### 1.4 AX 与二者的准确关系

AX 是 Agent Executor。它关心的是一个 conversation 由哪些 execution turn 构成、输入输出如何写入 durable event log、用哪个 Harness 执行、断开或失败后如何继续，而不是 Pod、network namespace、cgroup、gVisor 或 VM 本身。[`AX README.md:12-29`](../../agent-substrate/ax/README.md#L12)、[`proto/ax.proto:27-44`](../../agent-substrate/ax/proto/ax.proto#L27)

AX 当前有两类执行方式：

1. 本地内置 Harness，在 AX/sidecar 进程所在机器直接执行，不要求 Kubernetes；
2. Substrate Harness：把一个 AX conversation 映射为同名 Substrate Actor，执行一轮时 `CreateActor -> ResumeActor -> HarnessService.Connect`，结束时 `SuspendActor` 归还 Worker。

AX 当前**没有** Agent Sandbox provider。架构上可以新增，但需要实现 Claim/create、Ready 等待、HarnessService 网络可达、suspend/delete、snapshot 与 conversation identity 对齐等逻辑；这些都不是现有功能。

### 1.5 一个重要的版本兼容陷阱

AX HEAD 的 [`go.mod:11`](../../agent-substrate/ax/go.mod#L11) 固定 Substrate `3cb7433bd8a8`，并非本次本地 Substrate HEAD `cbdeb7d...`。AX adapter 从 `ResumeActor` 返回的 `AteomPodIp` 直接连接 `workerIP:80`。[`substrate.go:98-123`](../../agent-substrate/ax/internal/harness/substrate/substrate.go#L98)

但 Substrate HEAD 已改为：外部流量必须经 `atenet-router -> worker atunnel:443(mTLS) -> Actor:80`，旧的 Worker Pod `:80` 直连被移除并有 E2E 测试确认应被阻止。[`extproc.go:179-205`](../../agent-substrate/substrate/cmd/atenet/internal/router/extproc.go#L179)、[`networking_test.go:98-120`](../../agent-substrate/substrate/internal/e2e/suites/networking/networking_test.go#L98)

这意味着：

- AX 与 Substrate 有真实集成；
- 但不能直接把 AX HEAD、Substrate HEAD 当作已验证兼容组合；
- 部署 AX 时应按其部署文档把 AX pin、本地 Substrate checkout、集群 control plane 和 ateom image 对齐。[`AX manifests/README.md:175-193`](../../agent-substrate/ax/manifests/README.md#L175)

## 2. 调研范围与证据边界

### 2.1 固定版本

| 项目 | 固定版本 | 上游源码 |
|---|---|---|
| Agent Sandbox | `ac67d375a8b00c305a8647cdfa4aa876f43521e7` | [kubernetes-sigs/agent-sandbox@ac67d37](https://github.com/kubernetes-sigs/agent-sandbox/tree/ac67d375a8b00c305a8647cdfa4aa876f43521e7) |
| AX | `f327e23b5b842e9b700675ded9a6cdb79c505856` | [google/ax@f327e23](https://github.com/google/ax/tree/f327e23b5b842e9b700675ded9a6cdb79c505856) |
| Agent Substrate | `cbdeb7dbe003a55a16960a301bc595d9aa38b1ad` | [agent-substrate/substrate@cbdeb7d](https://github.com/agent-substrate/substrate/tree/cbdeb7dbe003a55a16960a301bc595d9aa38b1ad) |
| AX pin 的 Substrate | `3cb7433bd8a81a1f9aa54ef98418fbde8205fc42` | [agent-substrate/substrate@3cb7433](https://github.com/agent-substrate/substrate/tree/3cb7433bd8a81a1f9aa54ef98418fbde8205fc42) |

### 2.2 “已实现”“文档声明”“路线图”分开判断

本文采用以下证据顺序：

1. 可执行源码与 CRD schema；
2. 部署清单和 SDK 调用；
3. README/architecture 对设计意图的说明；
4. roadmap 只表示计划，不算当前实现；
5. blog 用于说明产品背景，不替代源码证据。

需要特别保守的地方：

- AX README 明确标记 active early development。[`AX README.md:3-10`](../../agent-substrate/ax/README.md#L3)
- Substrate README 明确说 early development、not ready for production；其 architecture 也声明大量内容仍是 aspirational。[`Substrate README.md:48-53`](../../agent-substrate/substrate/README.md#L48)、[`docs/architecture.md:1-4`](../../agent-substrate/substrate/docs/architecture.md#L1)
- Agent Sandbox roadmap 中的 auto suspend/resume、multi-sandbox per Pod、1000+ claims/s 等是计划，不应写成已完成能力。[`roadmap.md:15-22`](../../agent-substrate/agent-sandbox/roadmap.md#L15)、[`roadmap.md:41-54`](../../agent-substrate/agent-sandbox/roadmap.md#L41)
- 本次没有复现实测，所以本文不把 `sub-second`、`30x+` 或 roadmap 中的吞吐数字当作独立验证的 benchmark 结果。

## 3. 三者整体功能栈

~~~mermaid
flowchart TB
    User[Agent developer / client]

    subgraph AX[AX：Agent执行协议层]
      AXS[AX Server / CLI-local Controller]
      EV[(SQLite / Postgres<br/>Conversation Event Log)]
      HR[Harness Registry]
    end

    subgraph Compute[Compute backend：两条当前/潜在路径]
      SUB[Agent Substrate<br/>已实现AX adapter]
      AS[Agent Sandbox<br/>当前没有AX adapter]
    end

    subgraph K8S[Kubernetes infrastructure]
      WP[Substrate WorkerPool / Worker Pods]
      SCR[Sandbox CR / Pod / PVC / Service]
      RT[RuntimeClass<br/>gVisor / Kata]
    end

    User --> AXS
    AXS <--> EV
    AXS --> HR
    HR -->|Create/Resume/Suspend Actor| SUB
    HR -.->|未来需新增provider| AS
    SUB --> WP
    AS --> SCR
    SCR --> RT
~~~

这张图有三个必须注意的方向：

- `AX -> Substrate` 是实际代码依赖；
- `AX -.-> Agent Sandbox` 只是可能扩展，不是当前实现；
- `Agent Sandbox` 与 `Substrate` 是并列的 compute/lifecycle 方案，不应画成实线的上下层依赖。

## 4. Agent Sandbox 源码架构

### 4.1 核心定位：为 singleton stateful Pod 提供标准 Sandbox CRD

Agent Sandbox 不是新的 VMM。它给 Kubernetes 增加一个面向 agent sandbox 的声明式资源：

- 一个长期稳定的 `Sandbox` identity；
- 一个 backing Pod；
- 可选 PVC templates；
- 可选 headless Service 和稳定 FQDN；
- `Running/Suspended` desired mode；
- shutdown time / retain-or-delete policy；
- 由标准 `PodSpec.runtimeClassName` 选择 gVisor、Kata 等隔离 runtime。

核心 API 的共享蓝图 `SandboxBlueprint` 包括完整 `PodSpec`、PVC templates 和可选 Service；运行期 `SandboxSpec` 再加入 lifecycle 与 `operatingMode`。[`sandbox_types.go:136-221`](../../agent-substrate/agent-sandbox/api/v1beta1/sandbox_types.go#L136) `SandboxStatus` 暴露 Service/FQDN、conditions、Pod IP和 Node name。[`sandbox_types.go:250-277`](../../agent-substrate/agent-sandbox/api/v1beta1/sandbox_types.go#L250)

因此当前运行单位应理解为：

~~~text
1 Sandbox CR
  ├─ 0或1 backing Pod（Suspended时为0）
  │   └─ 1..N containers（仍属于同一个Sandbox）
  ├─ 0..N PVC
  └─ 0或1 headless Service
~~~

Roadmap 把“一个 Pod 内多个独立 sandbox”列为 Planned，说明当前不要把一个多容器 Pod误读为多个独立 Sandbox。[`roadmap.md:19-22`](../../agent-substrate/agent-sandbox/roadmap.md#L19)

### 4.2 Core controller 的声明式 reconcile 路径

用户创建/修改 `Sandbox` 后，典型链路是：

~~~text
Client
  -> kube-apiserver / etcd: Sandbox desired state
  -> Sandbox controller informer/workqueue
  -> Reconcile(Sandbox)
      -> reconcile PVCs
      -> reconcile backing Pod
      -> reconcile optional Service
      -> update Sandbox.status.conditions / PodIPs / NodeName
  -> normal Kubernetes Pod scheduling
      -> kube-scheduler -> kubelet -> CRI -> RuntimeClass runtime -> CNI/CSI
~~~

`SandboxReconciler.Reconcile` 先读取 CR，随后 `reconcileChildResources` 分别协调 PVC、Pod、Service并更新状态。[`sandbox_controller.go:203-330`](../../agent-substrate/agent-sandbox/controllers/sandbox_controller.go#L203) Running 且 Pod 不存在时，controller 从 `PodTemplate.Spec` 和 PVC references 构造新 Pod、设置 owner reference并写回 Kubernetes API。[`sandbox_controller.go:1134-1213`](../../agent-substrate/agent-sandbox/controllers/sandbox_controller.go#L1134)

这意味着它完整继承 Kubernetes controller 的优点：

- API/RBAC/audit/namespace/policy 与 GitOps 原生兼容；
- reconcile 可重试，可由 desired state 修复丢失的 child object；
- 可直接使用 scheduler、HPA、cluster autoscaler、CNI、CSI、RuntimeClass 生态；
- 代价是 per-Sandbox object、watch、status、Pod scheduling 仍然进入 Kubernetes control plane。

### 4.3 Extensions：Template、WarmPool、Claim

扩展 CRD 的分工如下：

| 对象 | 作用 | 当前源码语义 |
|---|---|---|
| `SandboxTemplate` | 可复用 Sandbox blueprint 与安全策略 | Pod/PVC/Service template、claim 注入策略、共享 NetworkPolicy |
| `SandboxWarmPool` | 维持指定数量的预热 Sandbox | 创建完整 `Sandbox` CR；core controller 再创建完整 Pod |
| `SandboxClaim` | 申请一个 Sandbox | 优先 adopt warm Sandbox；池空时从 Template cold-create |

`SandboxTemplate` 不是 Substrate 的 `ActorTemplate`。Agent Sandbox 的 template controller 当前主要确保 template hash label并创建/更新共享 `NetworkPolicy`；它不会 cold boot 一个 Golden Sandbox，也不会生成内存 snapshot。[`sandboxtemplate_controller.go:52-188`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxtemplate_controller.go#L52)

WarmPool controller 的真实动作也需要特别说明：它并非直接创建“裸 Pod”，而是批量创建完整 `Sandbox` CR，给其设置 WarmPool owner reference；随后 core Sandbox controller 根据这些 CR 创建 Pod。[`sandboxwarmpool_controller.go:341-382`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxwarmpool_controller.go#L341)、[`sandboxwarmpool_controller.go:735-781`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxwarmpool_controller.go#L735)

Claim 热路径是：

~~~text
Create SandboxClaim
  -> kube-apiserver / etcd
  -> SandboxClaim reconciler
  -> 从controller内存WarmSandboxQueue取candidate
  -> Kubernetes GET确认Sandbox仍可adopt且优先Ready candidate
  -> 更新Claim annotation，记录assigned Sandbox name
  -> patch Sandbox：
       ownerRef: SandboxWarmPool -> SandboxClaim
       launch-type: warm
       更新identity label / metadata / network policy selector
  -> Sandbox本来的Pod保持运行
  -> 更新Claim status Ready / Sandbox name / Pod IP
~~~

对应源码中，fast path先尝试 existing/adoption，池中无候选才查 Template cold-create。[`sandboxclaim_controller.go:430-526`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go#L430) Candidate队列优先返回 Ready Sandbox；adoption用 optimistic lock记录 Claim assignment并完成 owner transfer。[`sandboxclaim_controller.go:902-1038`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go#L902)、[`sandboxclaim_controller.go:1054-1156`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go#L1054)

因此 warm claim 的优化点是：**把 Pod 创建、image pull、scheduler placement、kubelet/CRI/CNI 准备提前到 pool 维护阶段**。Claim 到来时仍会访问 Kubernetes API、更新 CR并经过 controller reconcile，但不必在关键路径重新启动 Pod。

还有两个实际约束：

- Claim 注入 `env` 会强制 cold start；
- Claim 提供 `volumeClaimTemplates` 也会强制 cold start，因为预热 Pod 没有这些定制 volume。[`sandboxclaim_types.go:108-136`](../../agent-substrate/agent-sandbox/extensions/api/v1beta1/sandboxclaim_types.go#L108)

WarmPool controller 为 burst 做了不少 Kubernetes-style 防抖：默认 pool size 1、单轮最多 create/delete 300、informer expectation bookkeeping、readiness grace、unschedulable hold/retry。这些说明它的优化仍发生在 CR/controller/watch 模型内部，而不是把动态 Sandbox 状态移到另一套数据库。[`sandboxwarmpool_types.go:39-77`](../../agent-substrate/agent-sandbox/extensions/api/v1beta1/sandboxwarmpool_types.go#L39)、[`configuration.md:5-12`](../../agent-substrate/agent-sandbox/docs/configuration.md#L5)

Adoption也不是一条原子数据库事务：controller先更新 Claim的 assigned-sandbox annotation，再用 optimistic-lock patch转移 Sandbox owner；源码还要用 authoritative API read和补偿分支处理中间失败与409 conflict。[`sandboxclaim_controller.go:978-1016`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go#L978)、[`sandboxclaim_controller.go:1149-1165`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go#L1149) 项目自己的 APF说明也承认，高Claim速率下 latency-critical adoption writes会与 warm-pool refill产生的 Sandbox/Pod/Service create及informer list/watch争抢 kube-apiserver seats。[`docs/apf-insulation.md:26-49`](../../agent-substrate/agent-sandbox/docs/apf-insulation.md#L26) 这是一条很直接的源码库内证据：WarmPool优化了时间位置，但没有消除 Kubernetes声明式状态同步成本。

### 4.4 Suspend 的真实语义：core 只删除 Pod

当 `spec.operatingMode=Suspended` 时，core controller 删除该 Sandbox 拥有的 Pod；Pod完全消失后把 Suspended condition置为 True。[`sandbox_controller.go:1012-1043`](../../agent-substrate/agent-sandbox/controllers/sandbox_controller.go#L1012)

它没有在删除前调用 gVisor checkpoint、VMM snapshot，也没有将 guest/process RAM 保存到通用 repository。因为 PVC reconciliation 与 Pod reconciliation 分开，Sandbox CR、PVC以及配置的 Service identity可继续保留。切回 Running 后，controller重新创建 Pod并挂回 PVC。

所以 core suspend 的状态保留边界是：

| 状态 | core suspend 后是否保留 |
|---|---|
| Sandbox identity / metadata | 是 |
| PVC 持久文件 | 是 |
| Pod object / Pod IP / Node placement | 否 |
| 容器 rootfs 非持久层 | 否，除非另有 runtime/cloud snapshot能力 |
| 进程、内存、连接 | 否 |
| CPU/内存占用 | Pod删除后释放 |

这也解释了 roadmap 把已完成能力准确写成 **Suspend/Resume (PVC-based)**，而把 auto suspend/resume 与 scale-to-zero快速恢复继续列为计划。[`roadmap.md:132-137`](../../agent-substrate/agent-sandbox/roadmap.md#L132)

### 4.5 GKE PodSnapshot：可以保留内存，但不是 OSS core 的通用实现

仓库内确实有一条保存内存状态的路径，但边界必须说清楚：

- 文档要求 GKE Autopilot、gVisor node pool以及 `podsnapshot.gke.io` CRD；
- 当前只有 Python `PodSnapshotSandboxClient` 封装，Go SDK没有；
- Python `SnapshotEngine` 创建 `PodSnapshotManualTrigger` Kubernetes custom object并等待外部 controller完成；真正执行 gVisor checkpoint/存储/restore 的 controller不在 Agent Sandbox core controller 中；
- `suspend(snapshot_before_suspend=True)` 先创建 snapshot，再 patch `operatingMode=Suspended`；`resume()` 再切回 Running并验证新 Pod是否从目标 snapshot恢复。

源码证据：[`snapshots/_index.md:10-38`](../../agent-substrate/agent-sandbox/site/content/docs/sandbox/snapshots/_index.md#L10)、[`snapshot_engine.py:93-184`](../../agent-substrate/agent-sandbox/clients/python/agentic-sandbox-client/k8s_agent_sandbox/gke_extensions/snapshots/snapshot_engine.py#L93)、[`sandbox_with_snapshot_support.py:163-259`](../../agent-substrate/agent-sandbox/clients/python/agentic-sandbox-client/k8s_agent_sandbox/gke_extensions/snapshots/sandbox_with_snapshot_support.py#L163)

因此最准确的表述是：

> Agent Sandbox 在 GKE 特定扩展组合下可以使用 gVisor PodSnapshot 保存 filesystem + memory/process state；OSS core `Sandbox` controller 自身只实现 PVC-based suspend，不能据此宣称任意 Kubernetes 上安装 Agent Sandbox就天然获得通用内存快照。

### 4.6 SDK、连接和安全边界

Agent Sandbox 已提供 Go和 Python client。client层在 workload image暴露对应 HTTP agent server 时，可提供 command execution、file read/write/list等高层能力，并支持 direct URL、Gateway发现或本地 port-forward 等连接策略。它们是在 Sandbox Pod 中服务之上的 SDK协议，不等于 Kubernetes core CRD原生提供 `exec`/filesystem RPC。

仓库中的 `sandbox-router` 也只是 request data plane：它根据 `X-Sandbox-*` header、Pod informer cache或Service DNS转发到已有 Pod。它明确不会创建或查询 `Sandbox`，目标不存在时返回错误，而不会像 Substrate `atenet`一样先触发 Resume。[`sandbox-router/README.md:7-25`](../../agent-substrate/agent-sandbox/sandbox-router/README.md#L7)、[`sandbox-router/proxy/resolve.go:66-124`](../../agent-substrate/agent-sandbox/sandbox-router/proxy/resolve.go#L66) 这与 roadmap仍把 auto suspend/resume列为 Planned相吻合。

隔离边界则仍由 PodSpec与集群提供：

- `runtimeClassName` 决定 gVisor/Kata等 runtime；
- template controller可共享管理 NetworkPolicy；
- Template创建路径会应用 secure defaults；
- Cluster RBAC、admission、CNI enforcement、Pod security仍是关键可信部分。

Agent Sandbox也可以运行 Firecracker microVM，但调用链是 `Sandbox controller -> Kubernetes Pod -> RuntimeClass kata-fc -> containerd/Kata shim -> Firecracker`，并非 controller直接调用 Firecracker API。仓库示例只是在 `PodSpec`中设置 `runtimeClassName: kata-fc`。[`firecracker-sandbox/README.md:1-9`](../../agent-substrate/agent-sandbox/examples/firecracker-sandbox/README.md#L1)、[`sandbox-firecracker.yaml:20-44`](../../agent-substrate/agent-sandbox/examples/firecracker-sandbox/sandbox-firecracker.yaml#L20) 因而 Agent Sandbox core没有自己的 Firecracker snapshot、virtio-balloon或vCPU hotplug控制面；这些能力是否存在取决于所选 Kata/Firecracker runtime及集群集成，不能从 Agent Sandbox CRD本身推导。

## 5. AX 源码架构

### 5.1 AX解决什么问题

AX 的核心对象不是 Sandbox，而是：

| 对象 | 含义 |
|---|---|
| `Conversation` | 跨多轮存在的逻辑会话 |
| `Execution` | 一次 turn；有 exec ID和执行状态 |
| `Harness` | Agent/planner执行边界，负责 Start/Queue/Run/Close |
| `ConversationEvent` | append到 event log 的输入、输出、状态事件 |

`Harness` interface要求 controller保证同一 conversation同一时刻最多一个 Execution；Execution暴露 `Queue/Run/Close`。[`harness.go:25-63`](../../agent-substrate/ax/internal/harness/harness.go#L25) 外部 gRPC `ExecutionService.Exec` 返回 server stream，Actor内的 `HarnessService.Connect` 则是 AX server与远端 Harness之间的协议。[`proto/ax.proto:92-149`](../../agent-substrate/ax/proto/ax.proto#L92)

AX server/controller大致执行：

~~~text
Client Exec(conversationID, inputs)
  -> AX server做conversation in-flight保护
  -> Controller扫描event log决定续接状态与harness
  -> Harness.Start(conversationID)
  -> append PENDING input event
  -> Execution.Queue + Run
  -> 每个Harness output append event后stream给client
  -> append COMPLETED
  -> Execution.Close
~~~

Event log默认可用 SQLite，也支持 Postgres。它持久化的是 conversation execution history，不是 VM/process snapshot。

### 5.2 AX本地模式不依赖 Kubernetes

当未启用 `AX_SUBSTRATE=1` 时，内置 Harness在本机运行：有的 fork Python sidecar，有的直接在当前 Go进程使用本地 filesystem/shell工具。因此“AX推荐部署在 Kubernetes/Substrate上”不等于“AX core必须依赖 Kubernetes”。AX README也明确称其 compute-agnostic。[`AX README.md:66-74`](../../agent-substrate/ax/README.md#L66)

这再次说明 AX不是 sandbox runtime：本地模式下所谓环境隔离取决于用户如何部署 AX进程，而不是 AX内部自动创建 namespace/cgroup/microVM。

### 5.3 AX 与 Substrate 的真实调用链

AX直接依赖 Substrate public proto，`internal/ate/client.go` import `ateapipb` 并封装 `CreateAtespace/CreateActor/ResumeActor/SuspendActor`。[`internal/ate/client.go:15-100`](../../agent-substrate/ax/internal/ate/client.go#L15)

一轮 execution 的 identity和调用关系为：

~~~mermaid
sequenceDiagram
    participant C as AX Client
    participant X as AX Server/Controller
    participant E as AX Event Log
    participant A as Substrate ateapi
    participant W as Worker ateom + AX Harness

    C->>X: Exec(conversation_id, inputs)
    X->>E: scan conversation history
    X->>A: CreateActor(name=conversation_id)
    Note over X,A: AlreadyExists视为正常续聊
    X->>A: ResumeActor
    A-->>X: Actor + assigned Worker IP
    X->>W: HarnessService.Connect / HarnessStart
    W-->>X: output stream
    X->>E: append output/completion
    X-->>C: stream output
    X->>A: SuspendActor on Execution.Close
~~~

准确身份关系是：

~~~text
AX conversation_id == Substrate Actor name
~~~

Actor的逻辑身份保持不变，但每轮 Resume后的 Worker Pod/IP可以变化。正常执行结束时 `Close` 调 `SuspendActor`，Substrate checkpoint后释放 Worker slot。[`substrate.go:86-132`](../../agent-substrate/ax/internal/harness/substrate/substrate.go#L86)、[`substrate.go:235-255`](../../agent-substrate/ax/internal/harness/substrate/substrate.go#L235)

AX部署清单也真实创建 Substrate对象：一个5-replica `WorkerPool`、两个 `ActorTemplate` 和 snapshot bucket设置；AX server自己是3-replica Kubernetes ReplicaSet。[`ax-deployment.yaml:30-119`](../../agent-substrate/ax/manifests/ax-deployment.yaml#L30)、[`ax-deployment.yaml:123-164`](../../agent-substrate/ax/manifests/ax-deployment.yaml#L123)

### 5.4 AX 并不直接提供哪些能力

当前 AX API不能等同于 E2B/Agent Sandbox式通用 sandbox API：

- 没有通用 Sandbox create/delete API；
- 没有自己实现的 Node daemon、namespace/cgroup/VMM；
- 没有通用 PTY、任意文件上传下载、任意端口 expose API；
- command/filesystem能力来自具体 Harness中的工具或 Sandbox image内服务；
- event log、Harness内部 trajectory/cursor/workspace、Substrate snapshot是不同持久化域，并非一个原子事务。

AX README宣称 single-writer、resumable stream和automatic recovery，但当前实现仍是早期骨架。例如 single-writer保护只在 AX server进程内，而部署清单有3个 replica；不能由此推导出跨副本 exactly-once execution。本文因此把 AX评价为“Agent execution/resumption协议层”，不把所有声明视为已经达到生产语义。

## 6. Agent Substrate 源码架构

### 6.1 核心思想：把 Actor lifecycle 与 Worker Pod lifecycle解耦

Substrate针对大量 bursty、长时间 idle 的有状态 agent workload：预先运行较少的 Worker Pod，让 Kubernetes只管理物理容量；大量 logical Actor存在专用状态库中，active时才占用 Worker，idle时变成 snapshot。[`Substrate README.md:8-16`](../../agent-substrate/substrate/README.md#L8)、[`docs/architecture.md:34-55`](../../agent-substrate/substrate/docs/architecture.md#L34)

配置面与动态面分离：

| 面 | 对象 | 状态系统 | 变化频率 |
|---|---|---|---|
| 基础设施/配置面 | `WorkerPool`、`ActorTemplate`、`SandboxConfig` | Kubernetes CRD/API | 低频 |
| Actor动态控制面 | `Actor`、`Worker assignment`、snapshot ref、Atespace | Redis/ValKey | 高频 |

Substrate文档明确说 dynamic records不是 Kubernetes对象，因为更新太频繁，不适合 etcd。[`docs/glossary.md:26-45`](../../agent-substrate/substrate/docs/glossary.md#L26) `CreateActor`只写一个 `STATUS_SUSPENDED` Actor record，并不创建 Pod。[`create_actor.go:87-123`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L87)

### 6.2 组件职责

| 组件 | 部署/基数 | 职责 |
|---|---|---|
| `atecontroller` | Kubernetes controller | 将 WorkerPool reconcile成 Deployment/Worker Pods；为 ActorTemplate生成golden snapshot |
| `ateapi` | 中心 gRPC control plane | Actor lifecycle、Redis state、Worker schedule、Resume/Suspend workflow |
| `atelet` | 每 Node一个 DaemonSet | node supervisor；准备OCI bundle/runtime asset、搬运snapshot、调用目标ateom |
| `ateom` | 每 Worker Pod一个 | 驱动 gVisor runsc或 Kata+Cloud Hypervisor的 Run/Restore/Checkpoint；承载atunnel |
| `atenet` | DNS + Envoy router | 按Actor identity解析和路由；请求到达时触发 Resume；可停泊等待容量 |
| Redis/ValKey | control-plane state store | Actor/Worker/assignment/version/snapshot metadata |
| GCS/S3 | snapshot object store | 保存可跨Node恢复的 snapshot bytes |

`WorkerPool` controller构造 Kubernetes Deployment，replicas决定 Worker Pod数，每个 Pod内运行 `ateom`并监听 atunnel 443。[`workerpool_apply.go:58-157`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go#L58) `atelet`恢复时并行下载 snapshot、准备 runtime assets与OCI bundle，然后通过以 Worker Pod UID命名的 Unix socket调用 `ateom.RestoreWorkload`。[`atelet/main.go:517-618`](../../agent-substrate/substrate/cmd/atelet/main.go#L517)、[`ateompath.go:43-55`](../../agent-substrate/substrate/internal/ateompath/ateompath.go#L43)

### 6.3 Worker Pod、ateom 与 Actor sandbox 的基数

当前支持模型是：

~~~text
1 Worker Pod
  └─ 1 ateom
      └─ 同时最多1个active Actor sandbox
          └─ Actor workload可含1..N application containers
~~~

Scheduler只选择 `worker.assignment == nil` 且 sandbox class/label/node约束匹配的 Worker。[`scheduling.go:89-125`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go#L89) Glossary也明确，一个 Worker同一时刻最多host一个 Actor，但随时间可复用于多个 Actor。[`docs/glossary.md:38-45`](../../agent-substrate/substrate/docs/glossary.md#L38)

因此 Substrate的“multiplexing”当前是**时间复用**而非一个 Worker Pod内同时跑几十个 Actor。它的密度来源是 idle Actor不占 Worker，而不是在一个 Pod里并发装箱很多互不相关的 sandbox。

### 6.4 Resume 热路径为什么绕过 per-Actor Kubernetes

~~~text
atenet / API client: ResumeActor
  -> ateapi acquire Actor lock
  -> Redis读取Actor
  -> scheduler从ready Worker records中找assignment=nil
  -> versioned update claim Worker
  -> Actor = RESUMING，并写入Worker/Pod IP
  -> direct gRPC到目标Node atelet
  -> atelet下载snapshot、准备bundle/assets
  -> Unix socket到目标Worker Pod内ateom
  -> runsc restore 或 Cloud Hypervisor restore
  -> readiness
  -> Actor = RUNNING
  -> atenet通过atunnel:443路由请求
~~~

这条路径不会为该 Actor创建 Pod，也不等待 kube-scheduler、kubelet、CNI重新为它建立外层 Worker network。Kubernetes此前已经为 WorkerPool准备好了 Pod/CNI/resource limits。

但“bypass Kubernetes”不等于完全不依赖 Kubernetes：

- Worker capacity仍是 Deployment/Pod；
- Worker扩缩容和故障替换仍由 Kubernetes完成；
- Worker Pod IP、CNI、node placement、Secret/cert投影仍来自 Kubernetes；
- ActorTemplate/WorkerPool/SandboxConfig还是 CRD；
- 新Worker cold capacity仍要承担标准 Pod启动成本。

### 6.5 ActorTemplate、Golden Actor与golden snapshot

Substrate `ActorTemplate` 不只是 Pod blueprint。atecontroller为每个新 Template：

1. 在系统 `ate-golden` Atespace创建一个普通 Golden Actor；
2. Resume它完成真正 cold boot和应用初始化；
3. 等 readyz，或在没有完整readyz时默认再 warm-up 20秒；
4. 调普通 `SuspendActor`路径生成 snapshot；
5. 把 snapshot name记录到 `ActorTemplate.status.goldenSnapshot`；
6. 普通 Actor第一次 Resume且没有自己的 snapshot时，从这个共享基线恢复。

对应状态机在 [`actortemplate_controller.go:38-170`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L38)。这和 Agent Sandbox `SandboxTemplate controller`只维护 template hash和共享 NetworkPolicy形成鲜明区别。

### 6.6 Snapshot 与网络唤醒

Substrate直接管理 sandbox checkpoint/restore：

- gVisor class使用 `runsc` checkpoint/restore保存process tree与filesystem相关状态；
- microVM class使用 Kata guest + Cloud Hypervisor，内存 snapshot可通过 userfaultfd OnDemand restore；
- atelet负责 GCS/S3或node-local snapshot数据移动；
- Suspend后清空 Worker assignment，Actor保留 snapshot reference；
- 请求到达 suspended Actor时，atenet先调 Resume，再通过 atunnel将流量送到新 Worker。

当前 atenet的数据路径是：

~~~text
Client
  -> Actor DNS / atenet Envoy
  -> ateapi ResumeActor if needed
  -> worker atunnel :443 mTLS
  -> Actor private veth :80
~~~

源码和 architecture均明确 Worker Pod `:80`不再是直接 ingress。[`docs/architecture.md:333-350`](../../agent-substrate/substrate/docs/architecture.md#L333)

还应注意：自动“请求触发 Resume”已进入 atenet路径，但通用“根据空闲自动 Suspend”并非完整内核能力；调用方需要显式 `SuspendActor`。AX恰好在每轮 Execution.Close中做了这个调用。

## 7. 三者之间的真实联系

### 7.1 代码依赖图

~~~mermaid
flowchart LR
    AX[google/ax]
    PIN[Substrate pinned commit<br/>3cb7433 / 2026-07-06]
    HEAD[Substrate local HEAD<br/>cbdeb7d / 2026-07-31]
    AS[kubernetes-sigs/agent-sandbox]

    AX -->|go.mod + ateapipb + Harness adapter| PIN
    PIN -.->|同一仓库后续演进| HEAD
    AS -.->|相近问题、共享K8s/gVisor/Kata理念<br/>当前无代码集成| HEAD
    AS -.->|未来可新增Harness provider<br/>当前无代码集成| AX
~~~

全仓依赖检查的结果是：

- AX `go.mod`和源码明确 import Substrate；
- Substrate README明确把 AX列为其上的 distributed agent runtime示例；
- Substrate实现自身 control plane不依赖 AX，依赖方向单向为 `AX -> Substrate`；
- Agent Sandbox仓库没有 AX/Substrate module或 `ateapi/atelet/ateom/ActorTemplate/WorkerPool`集成；
- AX/Substrate也没有 `sigs.k8s.io/agent-sandbox`、`SandboxClaim`、`SandboxWarmPool`等引用。

Substrate README对 AX 的表述可见 [`README.md:44-47`](../../agent-substrate/substrate/README.md#L44)。AX部署指南则明确要求目标集群预装兼容 Substrate。[`AX manifests/README.md:8-13`](../../agent-substrate/ax/manifests/README.md#L8)

### 7.2 为什么官方材料会同时谈 Agent Sandbox 和 Substrate

二者代表 Google/Kubernetes agent runtime方向中的两个互补设计点：

- Agent Sandbox先提供 vendor-neutral、Kubernetes-native的标准 `Sandbox` API和生态入口；
- Agent Substrate探索当逻辑 Actor规模、idle比例和激活QPS非常高时，如何不让每个 Actor都变成 Pod、如何把 snapshot restore和请求路由放到 Kubernetes旁边的专用 control plane；
- AX展示上层 Agent harness/execution runtime如何消费 Substrate这种可 suspend/resume compute substrate。

这是“同一问题域与路线图的联系”，不是当前三个 repo已经拼成一套发行产品。

### 7.3 同名概念不能直接等同

| 概念 | Agent Sandbox | Agent Substrate | AX |
|---|---|---|---|
| Template | `SandboxTemplate`：Pod/PVC/Service blueprint和NetworkPolicy | `ActorTemplate`：workload定义 + Golden Actor/snapshot | Harness配置/image通过Substrate template部署 |
| Warm pool | 预热完整 Sandbox CR + Pod | 预热外层 Worker Pod slot | 不自建pool，消费Substrate WorkerPool |
| Instance identity | Sandbox CR name/UID；Claim绑定Sandbox | `(Atespace, Actor name)` logical identity | Conversation ID；Substrate模式下复用为Actor name |
| Suspend | core删Pod保PVC；GKE扩展可先PodSnapshot | first-class checkpoint，释放Worker | 调用compute provider的Close；Substrate provider会SuspendActor |
| Resume | controller重建/恢复Pod | Actor restore到任意匹配Worker | 续接conversation并启动Harness execution |
| Snapshot template | core没有golden snapshot | Template controller主动烘焙golden snapshot | 不负责底层snapshot格式 |

## 8. Control plane 与 data plane 对比

### 8.1 Agent Sandbox

~~~text
Cluster control plane:
  kube-apiserver + etcd
  Sandbox/SandboxClaim/WarmPool/Template controllers
  kube-scheduler

Node actuation:
  kubelet -> CRI -> RuntimeClass runtime
  CNI / CSI

Request data plane:
  direct Pod / headless Service / Gateway / port-forward / sandbox router

State data plane:
  PVC/CSI
  optional GKE PodSnapshot service
~~~

### 8.2 Agent Substrate

~~~text
Coarse Kubernetes control plane:
  WorkerPool / ActorTemplate / SandboxConfig CRD
  atecontroller -> Deployment / Worker Pods

High-frequency Actor control plane:
  ateapi + Redis/ValKey
  Actor/Worker assignment + Resume/Suspend workflow

Node actuation:
  atelet -> ateom -> runsc or Kata/Cloud Hypervisor

Request data plane:
  atenet DNS/Envoy -> atunnel -> Actor

Snapshot data plane:
  atelet <-> GCS/S3/local snapshot
~~~

### 8.3 AX

~~~text
Execution control plane:
  AX server/controller
  conversation single-writer boundary
  Harness registry

Execution state:
  SQLite/Postgres event log

Compute actuation:
  local Harness, or Substrate ateapi

Agent data plane:
  client <-> AX Exec stream <-> HarnessService/model/tools
~~~

AX的 event log与 Substrate/Agent Sandbox的 filesystem/memory snapshot解决不同状态域：前者记录“发生过哪些 Agent事件”，后者保存“计算环境此刻长什么样”。生产系统通常同时需要两者，还要解决它们之间的提交顺序、重复执行和副作用幂等。

## 9. 生命周期对比

| 阶段 | Agent Sandbox cold | Agent Sandbox warm claim | Substrate Actor | AX on Substrate |
|---|---|---|---|---|
| Create | 创建Sandbox CR，controller创建Pod | 创建Claim，优先adopt ready Sandbox/Pod | 只创建SUSPENDED Actor record | conversation首次执行时幂等CreateActor |
| Placement | kube-scheduler | 已在pool预热时完成 | ateapi从ready Worker中claim | 由ResumeActor间接完成 |
| Runtime start | kubelet/CRI/RuntimeClass | Pod已运行 | ateom Run或Restore | Actor内启动AX HarnessService |
| Ready | Pod readiness/status reconcile | Claim镜像Sandbox Ready | runtime readiness后Actor RUNNING | gRPC health后开始Harness turn |
| Idle | Pod继续占CPU/RAM reservation | 同左 | 调用方Suspend后释放Worker | Execution.Close显式SuspendActor |
| Suspend | core删Pod，PVC保留 | 同左 | checkpoint + release Worker | 调Substrate SuspendActor |
| Resume | 新Pod重新schedule；PVC重挂 | 已被claim的Sandbox同core语义 | restore到可能不同Worker | 同名Actor Resume后续聊 |
| Memory restore | core无；GKE PodSnapshot扩展可有 | 同左 | gVisor/CH first-class snapshot | 由Substrate完成，AX只续接Harness |

### 9.1 三种“warm”的延迟终点不同

进行性能比较时必须统一终点：

~~~text
request accepted
  -> logical identity allocated
  -> compute slot assigned
  -> sandbox/runtime ready
  -> application/harness ready
  -> first useful instruction/response
~~~

- Agent Sandbox WarmPool把完整 Pod Ready提前，Claim延迟主要是 API/controller/adoption/status；
- Substrate WorkerPool只把外层 Pod/CNI提前，Actor snapshot仍要下载和 restore；
- AX还要加 event log扫描、Harness gRPC连接、模型/tool执行。

所以不能比较“SandboxClaim Ready”“ResumeActor返回”“AX首个token”三个不同终点后直接得出架构快慢。

## 10. Kubernetes依赖矩阵

| 依赖维度 | Agent Sandbox | Agent Substrate | AX local | AX on Substrate |
|---|---|---|---|---|
| 必须由Kubernetes部署 | 是 | 是 | 否 | 是 |
| 每个逻辑实例是K8s对象 | 是，Sandbox/Claim | 否，Actor在Redis | 否 | AX conversation不是K8s对象 |
| 每次cold create走kube-scheduler | 是 | 只在Worker扩容时 | 否 | 只在Worker扩容时 |
| 每次resume走kube-scheduler | core通常是；GKE restore仍创建Pod | 否 | 否 | 否 |
| 依赖Pod/CNI网络 | 是 | Worker层仍是 | 否 | 是 |
| 依赖K8s controller最终一致性 | per-Sandbox是 | 低频Worker/config是；Actor hot path不是 | 否 | compute capacity低频是 |
| 自己实现高频调度 | 否，复用K8s | 是，Actor->Worker | 不适用 | 通过Substrate |
| 自己实现checkpoint mover | core否 | 是 | 否 | 通过Substrate |

## 11. 能否组合，以及需要什么改造

### 11.1 AX on Agent Sandbox：可设计，但当前没有

可以为 AX 实现新的 `Harness` provider：

~~~text
AX conversationID
  -> create/reuse SandboxClaim
  -> wait Claim/Sandbox Ready
  -> 通过Gateway/direct/Service连接Sandbox内HarnessService
  -> Run turn
  -> turn结束选择：
       keep Running
       core suspend（仅保PVC）
       GKE PodSnapshot + suspend
       delete Claim/Sandbox
~~~

必须补齐的语义包括：

1. `conversationID <-> Claim/Sandbox`稳定映射与幂等创建；
2. warm adoption后如何把 per-conversation identity/config安全注入；
3. HarnessService可达方式，以及Pod重建后地址更新；
4. turn结束何时snapshot，snapshot失败时AX event是否提交；
5. core PVC resume与GKE memory resume如何选择；
6. Claim/Sandbox expiration和AX conversation retention如何协调；
7. AX多副本single-writer与Kubernetes object并发更新如何fence。

由于 Agent Sandbox Claim注入env/PVC会强制cold start，若 AX需要每conversation动态配置，还应尽量通过外部 metadata、late binding或可热更新的服务协议注入，否则容易失去 WarmPool优势。

### 11.2 用 Agent Sandbox WarmPool承载 Substrate Worker：技术上可能，价值有限

理论上可让一个 WarmPool Pod image运行 ateom，再把它注册为 Substrate Worker；但当前两套controller职责高度重叠：

- Agent Sandbox WarmPool管理完整 warm Pod库存；
- Substrate WorkerPool controller也管理完整 Worker Deployment/Pod库存；
- Substrate Worker record、pod UID socket、certificate、atelet发现和lifecycle均假设其 WorkerPool模型。

要组合需要重写 Worker provider/discovery/controller，而不是简单改一个 `runtimeClassName`。如果每次 Actor assignment还要创建/adopt SandboxClaim，就会把 Substrate刻意移出的 per-Actor Kubernetes控制路径重新引入，因此通常不合理。

### 11.3 把 ateom当作 Agent Sandbox RuntimeClass：当前也不成立

`ateom` 是 Worker Pod内的 lifecycle coordinator，通过 Unix socket接收 atelet指令并调用 runsc/Kata/Cloud Hypervisor。它不是实现 CRI RuntimeClass handler的 runtime shim，也不是 Agent Sandbox计划中的通用 backend proto。因此不能在 Agent Sandbox template中写 `runtimeClassName: ateom`就完成集成。

## 12. 如何做可复现的性能比较

三套系统可以分别部署，但应先解决版本和实验边界。

### 12.1 建议部署矩阵

| 方案 | 可部署性 | 注意事项 |
|---|---|---|
| OSS Agent Sandbox on kind/GKE | 仓库有install/kind/e2e路径 | core snapshot只有PVC；gVisor/Kata与CNI能力取决于集群 |
| GKE Agent Sandbox add-on | GKE托管 | API版本可能与OSS HEAD不同；PodSnapshot需GKE/gVisor前提 |
| Substrate standalone | 有GCP cluster setup/demo/e2e | early development；依赖Redis、object store、runtime patch/assets |
| AX + Substrate | 有manifest/install脚本 | 必须使用AX固定的Substrate版本，不能直接配本地HEAD |
| AX + Agent Sandbox | 当前不可直接部署 | 先实现provider和Harness networking |

### 12.2 应测的四类路径

1. **Cold create**：从请求到第一条 useful instruction；
2. **Warm allocation**：Agent Sandbox Claim adoption vs Substrate首次从golden snapshot Resume；
3. **Stateful resume**：Agent Sandbox GKE PodSnapshot vs Substrate Actor snapshot；
4. **Burst与饱和**：并发请求超过 warm pool/Worker数量时的吞吐、p50/p95/p99、失败率和排队行为。

建议统一记录：

- API accept、slot assigned、Pod/runtime ready、application ready、first instruction五个时间点；
- kube-apiserver/etcd/controller QPS和CPU；
- scheduler/kubelet/CNI事件；
- Redis/ateapi QPS；
- snapshot bytes、download time、restore time、faulted pages；
- idle CPU、RSS、Pod reservation和对象数量；
- 节点故障、controller重启、重复请求下的正确性。

### 12.3 最有价值的直接对照

若目标是验证 “beside Kubernetes” 是否减少 declarative tax，最公平的实验是：

~~~text
固定相同node/image/runtime/application/readiness

Agent Sandbox:
  N个预热Sandbox Pod -> N个并发Claim adoption

Substrate:
  N个预热Worker Pod -> N个并发Actor Resume

同时测：
  first instruction latency
  kube-apiserver写QPS/etcd writes/watch events
  controller CPU
  Redis/ateapi CPU
~~~

然后再分别增加 cold pool miss、snapshot restore和容量不足场景。单测热命中不能代表完整成本，因为它隐藏了 pool预热阶段的 Pod创建税。

## 13. 对系统设计的启示

### 13.1 标准API与高频control plane可以分层，而非二选一

Agent Sandbox的优势是 Kubernetes-native治理和可移植API；Substrate的优势是 logical Actor与physical Worker解耦。一个长期方向可以是：

- 用较低频的 Kubernetes CRD表达 template、tenant policy、capacity pool、runtime class；
- 用专用低延迟数据库和同步RPC处理每次 activation、assignment、snapshot和route；
- 不为每个长期存在但大部分时间idle的 logical agent制造一个常驻Pod；
- 同时提供Kubernetes可观测的聚合状态与审计接口。

### 13.2 必须把三类状态分开

三仓恰好暴露出 Agent系统至少三类状态：

| 状态域 | 例子 | 负责系统 |
|---|---|---|
| 语义执行状态 | prompt、message、tool call、execution status | AX event log |
| 计算状态 | process tree、memory、ephemeral rootfs | Substrate snapshot / GKE PodSnapshot |
| 持久数据状态 | workspace、dataset、outputs | PVC/DurableDir/object store |

只有 memory snapshot不能保证工具副作用exactly-once，只有 event log也不能恢复浏览器/解释器内存。设计时需要明确三个域的commit barrier、版本号和失败恢复策略。

### 13.3 “Suspend”必须带限定词

同一个词在三者中的资源/状态语义不同：

- Agent Sandbox core suspend = delete Pod + retain PVC identity；
- GKE Agent Sandbox snapshot suspend = PodSnapshot后delete Pod；
- Substrate suspend = checkpoint Actor sandbox + release Worker；
- AX suspend = compute provider在Execution.Close上的动作，当前Substrate provider调用SuspendActor。

论文或 benchmark必须写清楚是哪一种，否则会把“CPU/RAM释放”“进程内存保留”“PVC保留”混成同一能力。

## 14. 最终判断

### 14.1 Agent Sandbox

它是三者中最 Kubernetes-native 的标准化 Sandbox API：擅长把 singleton stateful Pod、PVC、Service、lifecycle、secure RuntimeClass和warm claim统一成 CRD/controller/SDK。WarmPool能把完整 Pod cold start移出Claim热路径，但每个 Sandbox仍是Kubernetes对象和Pod。OSS core suspend只保PVC；内存恢复依赖GKE PodSnapshot扩展。

### 14.2 Agent Substrate

它是三者中专门为大量 idle logical instance做 multiplexing的控制面：Kubernetes只管理较少的 Worker capacity，ateapi/Redis管理大量 Actor，atelet/ateom直接 checkpoint/restore，atenet在请求到达时唤醒和路由。它比 Agent Sandbox更彻底地绕过 per-Actor Pod生命周期，但也承担了自研调度、一致性、snapshot、HA、安全和可观测性的复杂度，目前仍属早期项目。

### 14.3 AX

它位于两者之上，负责 Agent conversation/execution/harness/event-log语义。当前已实现的集群 compute backend是 Substrate，而不是 Agent Sandbox；本地模式则无需 Kubernetes。AX + Substrate组合可表达“每轮唤醒同名Actor、执行Harness、结束后checkpoint归还Worker”，但版本必须按 AX pin对齐，durable execution语义也仍需继续完善。

### 14.4 一句话关系图

~~~text
                         已实现
AX conversation/harness ---------> Substrate logical Actor + snapshot Worker multiplexing
        |
        | 未来可新增provider，当前没有
        v
Agent Sandbox CRD + Claim/WarmPool + one Sandbox/one backing Pod

Agent Sandbox 与 Substrate：并列、相似目标、不同对象粒度和hot path；当前无代码依赖。
~~~

## 15. 源码索引与参考资料

### 15.1 Agent Sandbox

- 项目定位与架构：[`README.md`](../../agent-substrate/agent-sandbox/README.md)
- Core Sandbox API：[`api/v1beta1/sandbox_types.go`](../../agent-substrate/agent-sandbox/api/v1beta1/sandbox_types.go)
- Core controller：[`controllers/sandbox_controller.go`](../../agent-substrate/agent-sandbox/controllers/sandbox_controller.go)
- Claim API/controller：[`sandboxclaim_types.go`](../../agent-substrate/agent-sandbox/extensions/api/v1beta1/sandboxclaim_types.go)、[`sandboxclaim_controller.go`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxclaim_controller.go)
- Template API/controller：[`sandboxtemplate_types.go`](../../agent-substrate/agent-sandbox/extensions/api/v1beta1/sandboxtemplate_types.go)、[`sandboxtemplate_controller.go`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxtemplate_controller.go)
- WarmPool API/controller：[`sandboxwarmpool_types.go`](../../agent-substrate/agent-sandbox/extensions/api/v1beta1/sandboxwarmpool_types.go)、[`sandboxwarmpool_controller.go`](../../agent-substrate/agent-sandbox/extensions/controllers/sandboxwarmpool_controller.go)
- GKE PodSnapshot Python扩展：[`gke_extensions/snapshots`](../../agent-substrate/agent-sandbox/clients/python/agentic-sandbox-client/k8s_agent_sandbox/gke_extensions/snapshots)
- OSS/GKE add-on关系：[`anthropic-managed-agents/_index.md`](../../agent-substrate/agent-sandbox/site/content/docs/use-cases/anthropic-managed-agents/_index.md)
- Roadmap：[`roadmap.md`](../../agent-substrate/agent-sandbox/roadmap.md)

### 15.2 AX

- 项目定位：[`README.md`](../../agent-substrate/ax/README.md)
- API对象与stream：[`proto/ax.proto`](../../agent-substrate/ax/proto/ax.proto)
- Harness interface：[`internal/harness/harness.go`](../../agent-substrate/ax/internal/harness/harness.go)
- Substrate Harness adapter：[`internal/harness/substrate/substrate.go`](../../agent-substrate/ax/internal/harness/substrate/substrate.go)
- Substrate control client：[`internal/ate/client.go`](../../agent-substrate/ax/internal/ate/client.go)
- Kubernetes部署：[`manifests/ax-deployment.yaml`](../../agent-substrate/ax/manifests/ax-deployment.yaml)、[`manifests/README.md`](../../agent-substrate/ax/manifests/README.md)

### 15.3 Agent Substrate

- 项目定位：[`README.md`](../../agent-substrate/substrate/README.md)
- Architecture：[`docs/architecture.md`](../../agent-substrate/substrate/docs/architecture.md)
- Glossary与基数：[`docs/glossary.md`](../../agent-substrate/substrate/docs/glossary.md)
- CreateActor：[`cmd/ateapi/internal/controlapi/create_actor.go`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go)
- Actor scheduler：[`cmd/ateapi/internal/scheduling/scheduling.go`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go)
- Resume workflow：[`cmd/ateapi/internal/controlapi/workflow_resume.go`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go)
- WorkerPool controller：[`cmd/atecontroller/internal/controllers/workerpool_apply.go`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go)
- ActorTemplate golden snapshot controller：[`cmd/atecontroller/internal/controllers/actortemplate_controller.go`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go)
- Node restore路径：[`cmd/atelet/main.go`](../../agent-substrate/substrate/cmd/atelet/main.go)
- Current ingress path：[`cmd/atenet/internal/router/extproc.go`](../../agent-substrate/substrate/cmd/atenet/internal/router/extproc.go)

### 15.4 外部资料

- [Kubernetes SIG Apps Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
- [Agent Sandbox documentation](https://agent-sandbox.sigs.k8s.io/docs/)
- [GKE Agent Sandbox and Agent Substrate announcement](https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate)
- [Google AX](https://github.com/google/ax)
- [Agent Substrate](https://github.com/agent-substrate/substrate)
- [Agent Executor announcement](https://cloud.google.com/blog/products/ai-machine-learning/agent-executor-googles-distributed-agent-runtime)

### 15.5 本系列既有 Survey

- [`20260801T081103Z-google-ax-agent-substrate-vs-agentenv-survey.md`](./20260801T081103Z-google-ax-agent-substrate-vs-agentenv-survey.md)
- [`20260803T121206Z-agent-substrate-kubernetes-declarative-tax-vs-akernel-survey.md`](./20260803T121206Z-agent-substrate-kubernetes-declarative-tax-vs-akernel-survey.md)
- [`20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md)
- [`20260804T111428Z-agent-substrate-ateapi-atelet-ateom-worker-sandbox-cardinality-survey.md`](./20260804T111428Z-agent-substrate-ateapi-atelet-ateom-worker-sandbox-cardinality-survey.md)
