# Agent Substrate 的 ateapi、atelet、ateom、Worker Pod 与 Actor Sandbox：组件职责和运行时基数源码分析

> 调研时间：2026-08-04 19:14:28（Asia/Shanghai）  
> 文档时间戳：20260804T111428Z  
> Agent Substrate commit：`cbdeb7dbe003a55a16960a301bc595d9aa38b1ad`  
> 方法：以固定commit源码判断当前实现，以`docs/glossary.md`、`docs/architecture.md`和`docs/api-guide.md`说明设计意图；若注释、设计文档与可执行路径存在差异，分别注明。本次没有部署集群或运行benchmark。

## 1. 结论摘要

### 1.1 控制与执行组件的最短定义

| 组件 | 部署位置 | 核心职责 | 不负责什么 |
|---|---|---|---|
| `ActorTemplate controller` | `atecontroller`中的Kubernetes controller | 为每个ActorTemplate编排Golden Actor的cold boot、warm-up、Suspend并记录golden snapshot | 不直接执行runsc/VM checkpoint，不为每个普通Actor创建Pod |
| `ateapi` | Kubernetes中的中心控制面服务 | Actor API、Redis状态、Actor→Worker调度、Resume/Suspend workflow | 不直接启动runsc/VM，不准备node文件和snapshot bytes |
| `atelet` | 每个Node一个DaemonSet | node-level supervisor；准备OCI bundle/runtime assets/volume，搬运snapshot，找到目标ateom | 不做全局Actor placement，不直接承载Agent应用请求 |
| `ateom` | 每个Worker Pod内部一个 | 驱动gVisor或Cloud Hypervisor，Run/Restore/Checkpoint一个Actor workload；承载atunnel | 不选择Actor放到哪个Worker，不是全局snapshot store |

一次典型Resume路径是：

~~~text
atenet / client
  → ateapi
      → Redis lock + Actor/Worker records
      → 从空闲Worker cache选择一个Worker Pod
      → 写入Worker.assignment和Actor physical location
      → direct gRPC atelet（目标Node）
          → snapshot/runtime asset/OCI bundle准备
          → 按Worker Pod UID打开Unix socket
          → ateom.RestoreWorkload
              → runsc restore 或 Cloud Hypervisor restore
              → readiness
              → atunnel激活当前Actor
~~~

### 1.2 Worker Pod与内部sandbox的准确关系

“最小运行时单位”必须按维度回答：

| 维度 | 单位 |
|---|---|
| Kubernetes placement、CNI和长期预热容量 | Worker Pod |
| Agent Substrate logical lifecycle、snapshot和路由identity | Actor |
| 不可信Agent代码的安全执行边界 | Worker Pod内部的gVisor sandbox或Cloud Hypervisor microVM |
| 当前Worker capacity slot | 一个Worker Pod = 一个并发Actor slot |

因此可以把Worker Pod理解为一个已经完成Kubernetes调度和CNI setup的**外层warm carrier/physical slot**；真正的Agent应用运行在Pod内部由ateom创建或恢复的sandbox中。但不能因此说Worker Pod与capacity无关：当前调度模型里一个Worker Pod就是一个并发slot。

### 1.3 一个ateom能否启动多个sandbox

准确答案分为三种：

1. **同时承载多个Actor sandbox：当前受支持模型不允许。**一个Worker record只有一个`assignment`，scheduler只选择`assignment == nil`的Worker，atunnel也只允许一个active Actor。
2. **先后运行不同Actor sandbox：允许。**Actor A Suspend/Checkpoint后释放Worker；同一个Worker Pod/ateom回到available，之后可以Run/Restore Actor B。
3. **一个Actor workload内部运行多个container：允许。**这些container仍属于同一个Actor sandbox，不是多个Actor sandbox。

最简基数是：

~~~text
1 Kubernetes Worker Pod
  └─ 1 ateom
      └─ 同时最多1个active Actor sandbox
          └─ 1..N application containers
~~~

### 1.4 Golden Actor与Golden Snapshot

每个新ActorTemplate由ActorTemplate controller创建一个位于系统`ate-golden` Atespace的普通Golden Actor，在现有Worker上cold boot该模板，等待readyz或默认20秒warm-up，再调用普通Suspend路径生成模板级共享Golden Snapshot。普通Actor第一次Resume时若没有自己的snapshot，默认从这份共享基线恢复；产生自己的Latest Snapshot后，后续Resume优先恢复自己的状态。

## 2. 官方文档在哪里解释这些组件

### 2.1 最适合快速查定义：Glossary

[`docs/glossary.md`](../../agent-substrate/substrate/docs/glossary.md)是最直接的入口：

- Actor是Suspend/Resume和跨Worker移动的logical unit；
- Worker record对应一个Worker Pod；
- Worker同一时刻最多host一个Actor，但一段时间内可复用给多个Actor；
- ateapi拥有Actor lifecycle、调度和snapshot coordination；
- atelet是DaemonSet node supervisor；
- ateom运行在每个Worker Pod内部并驱动sandbox runtime。

关键证据：[`docs/glossary.md:38`](../../agent-substrate/substrate/docs/glossary.md#L38)、[`docs/glossary.md:43`](../../agent-substrate/substrate/docs/glossary.md#L43)、[`docs/glossary.md:49`](../../agent-substrate/substrate/docs/glossary.md#L49)、[`docs/glossary.md:56`](../../agent-substrate/substrate/docs/glossary.md#L56)、[`docs/glossary.md:60`](../../agent-substrate/substrate/docs/glossary.md#L60)。

### 2.2 最适合看组件关系：Architecture

[`docs/architecture.md`](../../agent-substrate/substrate/docs/architecture.md)的System Components章节解释：

- ateapi由state store、scheduler和workflow engine组成；
- atelet与ateom构成node supervisor；
- atelet管理物理Worker Pods并移动snapshot；
- ateom分为`ateom-gvisor`和`ateom-microvm`；
- gVisor通过runsc checkpoint/restore；
- microVM通过Kata + Cloud Hypervisor执行和恢复。

关键证据：[`docs/architecture.md:298`](../../agent-substrate/substrate/docs/architecture.md#L298)、[`docs/architecture.md:313`](../../agent-substrate/substrate/docs/architecture.md#L313)、[`docs/architecture.md:325`](../../agent-substrate/substrate/docs/architecture.md#L325)。

### 2.3 已有中文架构Survey

三套系统横向Survey中的以下章节也解释了这些组件：

- [`5.3 自研高频Actor control plane`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md#L491)；
- [`5.4 Worker、atelet和ateom`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md#L551)；
- [`5.5 Actor request data plane`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md#L584)；
- [`5.6 Actor lifecycle路径`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md#L621)。

本文在此基础上进一步回答Worker和sandbox的基数问题。

### 2.4 Golden Snapshot的官方说明入口

[`docs/api-guide.md`](../../agent-substrate/substrate/docs/api-guide.md)的Operational Workflow描述Golden Snapshot与普通Actor resumption；[`docs/glossary.md`](../../agent-substrate/substrate/docs/glossary.md)区分模板级Golden Snapshot和每Actor Last Snapshot。当前源码与API Guide在“temporary Golden Pod”和`boot`参数范围上存在差异，本文第4节按源码给出限定。

## 3. 完整层级：Kubernetes capacity、Worker slot和Actor execution

~~~text
Kubernetes Cluster
│
├─ Agent Substrate control plane
│   ├─ ateapi
│   ├─ Redis / ValKey
│   ├─ atecontroller
│   └─ atenet / Envoy
│
├─ Node A
│   ├─ atelet DaemonSet Pod                 1 per Node
│   │
│   ├─ Worker Pod A1                        1 warm slot
│   │   ├─ ateom-gvisor or ateom-microvm    1 per Worker Pod
│   │   ├─ atunnel                          ateom内嵌/同进程子系统
│   │   └─ Actor X sandbox                  0 or 1 active Actor
│   │       ├─ pause/sandbox infrastructure
│   │       └─ application container(s)
│   │
│   └─ Worker Pod A2                        另一个warm slot
│       ├─ ateom
│       └─ available，等待assignment
│
└─ Node B
    ├─ atelet
    └─ Worker Pods ...
~~~

这里有四层identity：

1. `ActorRef = (atespace, actor name)`：稳定logical identity；
2. Actor UID/version：某个Actor资源incarnation与并发控制；
3. Worker record：Worker Pool中某个physical slot的控制面投影；
4. Worker Pod UID：atelet用来定位具体ateom socket和node-local资源。

Actor可以跨Resume被放到不同Worker；Worker Pod保持存在，并在不同时段承载不同Actor。

WorkerPool controller生成的Worker Pod模板当前只有一个名为`ateom`的外层container；Actor的application containers不是与ateom并列的Kubernetes sidecars，而是ateom在内层gVisor/microVM中创建的workload。证据：[`workerpool_apply.go:58`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go#L58)、[`workerpool_apply.go:138`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go#L138)。

## 4. ActorTemplate controller、Golden Actor与Golden Snapshot

### 4.1 先修正“Resume/Warm/Suspend”的准确含义

本文相关架构描述中使用过“ActorTemplate controller通过Golden Actor的Resume/Warm/Suspend生成golden snapshot”。当前源码中没有`WarmActor` RPC或独立Warm lifecycle。更准确的执行链是：

~~~text
CreateActor(Golden Actor)
  → ResumeActor
      → 因模板尚无golden snapshot而cold boot
      → RunWorkload等待配置了readyz的containers返回200
  → 如果并非所有containers都有readyz，再等待默认20秒warm-up
  → SuspendActor
      → ateapi/atelet/ateom执行checkpoint与上传
  → ActorTemplate.status.goldenSnapshot = snapshot name
  → ActorTemplate.status.phase = Ready
~~~

所以这里的“Warm”只是让application完成昂贵初始化并稳定下来：有完整readyz时由`ResumeActor`的ready gate完成，没有完整readyz时使用20秒wall-clock fallback。它不是第四种RPC。

### 4.2 ActorTemplate controller是什么

`ActorTemplate`是Kubernetes CRD，定义一类Actor的不可变workload blueprint，包括：

- pause image与application containers；
- environment和volume；
- gVisor或microVM sandbox class；
- Worker selector；
- snapshot location；
- `onPause`与`onCommit` snapshot scope；
- container readyz。

ActorTemplate controller的实现是`ActorTemplateReconciler`。它通过controller-runtime watch `ActorTemplate`对象，读取和更新Kubernetes CRD status，同时通过Substrate `ControlClient`调用ateapi。证据：[`actortemplate_controller.go:47`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L47)、[`actortemplate_controller.go:65`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L65)、[`actortemplate_controller.go:190`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L190)。

#### 为什么它叫Kubernetes controller

Kubernetes controller不是某个固定binary的名字，而是一类持续运行的control loop：

~~~text
watch API objects
  → 读取desired state（通常在spec）
  → 观察current state（status或外部系统）
  → 执行动作缩小两者差距
  → 更新status
  → 等待下一次事件或定时requeue
~~~

Kubernetes内置的Deployment、ReplicaSet和Node controllers属于这类；第三方也可以为CRD实现custom controller。ActorTemplate controller之所以是Kubernetes custom controller，是因为它具备完整的controller-runtime结构：

1. `ActorTemplate`由CRD扩展进Kubernetes API；
2. `SetupWithManager`通过`For(&ActorTemplate{})`注册watch；
3. 对象变化后，controller-runtime把namespace/name包装成`ctrl.Request`调用`Reconcile`；
4. Reconcile使用Kubernetes client `Get`读取ActorTemplate；
5. 它根据`status.phase`选择下一步动作；
6. 通过`Status().Update`写回observed state；
7. 需要等待时返回`RequeueAfter`，失败时返回error，由controller-runtime再次reconcile。

它是**运行在Kubernetes上的自定义controller**，不是编译进`kube-controller-manager`的内置controller。`atecontroller`这个Pod/binary可以同时注册多个Reconciler，例如ActorTemplate、WorkerPool和NetworkPolicy controller。

Custom controller也不要求所有动作都是创建Kubernetes Pod。它可以把Kubernetes CRD作为低频desired-state API，再调用外部系统。ActorTemplate controller调用的外部actuator正是ateapi：

~~~text
Kubernetes API中的ActorTemplate spec/status
          │ watch/reconcile
          ▼
ActorTemplateReconciler（atecontroller）
          │ gRPC CreateActor/ResumeActor/SuspendActor
          ▼
ateapi + Redis + Worker/atelet/ateom
~~~

所以这里存在两个协作控制面：Kubernetes保存ActorTemplate这个声明式对象及其status，Substrate专用控制面执行高频Actor lifecycle。ActorTemplate controller是两者之间的桥。

它不等于：

- kube-apiserver：负责Kubernetes API、认证、admission和etcd persistence；
- kube-scheduler：给Kubernetes Pod选择Node；
- kubelet：在Node上物化Pod；
- ateapi scheduler：给logical Actor选择已有Worker slot。

对ActorTemplate controller而言，desired state可以概括为“该Template拥有可用的golden snapshot并进入Ready”；observed state由`status.phase/goldenActorID/goldenSnapshot/conditions`和ateapi调用结果表示。它用显式phase把长流程拆到多次Reconcile，而不是在一次Reconcile中阻塞等待完整cold boot、warm-up和snapshot。

它和WorkerPool controller的职责不同：

| Controller | 输入 | 主要输出/动作 |
|---|---|---|
| WorkerPool controller | WorkerPool CRD | Kubernetes Deployment和长期Worker Pods |
| ActorTemplate controller | ActorTemplate CRD | 一次Golden Actor初始化流程、golden snapshot引用和Template Ready状态 |

ActorTemplate controller本身不下载snapshot、不调用runsc，也不直接执行VM snapshot；它只驱动ateapi暴露的普通Actor lifecycle。

### 4.3 Controller状态机

ActorTemplate status定义四个当前使用阶段：

~~~text
Initial
  → ResumeGoldenActor
  → WaitGoldenActor
  → Ready
~~~

类型和status字段见[`actortemplate_types.go:22`](../../agent-substrate/substrate/pkg/api/v1alpha1/actortemplate_types.go#L22)、[`actortemplate_types.go:361`](../../agent-substrate/substrate/pkg/api/v1alpha1/actortemplate_types.go#L361)。

#### Initial：创建Golden Actor

Controller首先确保系统保留Atespace `ate-golden`存在，然后创建一个普通Substrate Actor record：

~~~text
atespace              = ate-golden
actor name            = string(ActorTemplate.metadata.uid)
actor template ref    = <ActorTemplate namespace>/<ActorTemplate name>
initial Actor status  = SUSPENDED（由通用CreateActor路径设置）
~~~

随后把`ActorTemplate.status.goldenActorID`设为该Template UID，并进入`ResumeGoldenActor`。源码见[`actortemplate_controller.go:80`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L80)；系统Atespace常量见[`internal/resources/actor.go:24`](../../agent-substrate/substrate/internal/resources/actor.go#L24)。

#### ResumeGoldenActor：第一次cold boot

Controller调用普通`ResumeActor`，请求中没有特殊“golden boot”runtime参数：[`actortemplate_controller.go:118`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L118)。

此时Golden Actor没有自己的`LatestSnapshot`，ActorTemplate也尚未写入`status.goldenSnapshot`。ateapi的Resume选择逻辑因此落到cold boot分支：

~~~text
ActorTemplate spec
  → 选择一个assignment为空且sandbox class匹配的普通Worker
  → atelet准备OCI/runtime assets
  → ateom RunWorkload
  → application初始化
~~~

对应snapshot选择与cold boot源码：[`workflow_resume.go:70`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L70)、[`workflow_resume.go:502`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L502)。

#### WaitGoldenActor：ready gate或20秒warm-up

`RunWorkload`会等待所有声明了readyz的container就绪。当ActorTemplate的每个container都有readyz时，`ResumeActor`成功已经意味着全部probe返回HTTP 200，controller把额外warm-up设为0。

如果没有container或任一container没有readyz，controller使用默认20秒等待，作为“给workload时间完成初始化”的fallback。证据：[`actortemplate_controller.go:35`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L35)、[`actortemplate_controller.go:138`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L138)、[`actortemplate_controller.go:195`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L195)。

这意味着没有readyz时，controller并不知道application是否真的完成初始化；20秒只是启发式等待，不是application-level correctness barrier。

#### Ready：Suspend并记录共享snapshot

等待结束后，controller调用普通`SuspendActor(Golden Actor)`。Suspend成功响应中的`Actor.latestSnapshot`必须非空；controller把snapshot name写入`ActorTemplate.status.goldenSnapshot`，设置Phase=`Ready`和Ready condition。证据：[`actortemplate_controller.go:145`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L145)。

Phase Ready后，当前controller不再周期刷新或自动重建golden snapshot；ActorTemplate spec本身被设计为immutable，升级application通常应创建新Template。

### 4.4 Golden Actor到底是什么

Golden Actor是某个ActorTemplate的一次**种子/构建Actor**，目的类似用镜像配方启动builder instance：

| 概念 | 类比 | 生命周期作用 |
|---|---|---|
| ActorTemplate | image/build recipe或class | 定义code、runtime与snapshot policy |
| Golden Actor | builder instance | 运行一次模板，完成昂贵初始化 |
| Golden Snapshot | pre-initialized base image/checkpoint | 作为普通Actor的共享初始恢复点 |
| 普通Actor | 从base派生的instance | 恢复后形成自己的state和snapshot lineage |

Golden Actor没有特殊sandbox runtime。它仍然：

- 使用普通Actor record和status；
- 由ateapi的相同scheduler选择Worker；
- 占用一个正常Worker slot；
- 经相同atelet/ateom执行Run和Checkpoint；
- Suspend后释放Worker assignment。

特殊之处只有系统identity和用途：它位于`ate-golden`，名字取ActorTemplate UID，并由controller用来制作模板基线。

### 4.5 当前实现没有创建独立Golden Pod

`docs/api-guide.md`把这一步写成“Substrate starts a temporary Golden Pod”，并描述为gVisor制作“Version 0”。但当前`ActorTemplateReconciler`源码没有创建Pod或Deployment；它调用`CreateActor`和`ResumeActor`。Golden Actor随后被调度到WorkerPool中已经存在的普通Worker Pod，并在其内部启动gVisor/microVM sandbox。当前`ActorSnapshot`也没有显式`version=0`字段，“Version 0”更适合作为“模板初始基线”的概念称呼，而不是持久schema字段。

因此更准确的当前实现表述是：

> temporary Golden Actor execution on a regular Worker Pod，而不是为每个ActorTemplate创建一个专用Golden Kubernetes Pod。

文档措辞见[`docs/api-guide.md:236`](../../agent-substrate/substrate/docs/api-guide.md#L236)，源码路径见[`actortemplate_controller.go:96`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L96)、[`actortemplate_controller.go:130`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L130)。如果WorkerPool本身容量不足，后台HPA/Kubernetes可能另行扩Worker Pods，但这不是controller创建per-template Golden Pod。

Golden Actor的**运行实例**是临时的；其logical Actor record在Suspend后仍保持SUSPENDED。当前controller完成后没有调用DeleteActor，ActorTemplate deletion分支也直接返回，没有在本controller中清理Golden Actor或snapshot。证据：[`actortemplate_controller.go:75`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L75)、[`actortemplate_controller.go:183`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L183)。

### 4.6 Golden Snapshot是什么、保存在哪里

Golden Snapshot是Golden Actor完成初始化后，由Suspend生成的一份**模板级共享ActorSnapshot**。它不是ActorTemplate YAML，也不是Kubernetes VolumeSnapshot。

实际链路是：

~~~text
ActorTemplate controller
  → ateapi SuspendActor
      → Worker上的Actor进入SUSPENDING
      → atelet.Checkpoint
          → ateom-gvisor: runsc checkpoint/fscheckpoint
          → ateom-microvm: CH pause + sparse memory snapshot
      → atelet上传snapshot artifacts/manifest到GCS或S3
      → ateapi在Redis/ValKey创建ActorSnapshot metadata + physical location
      → Golden Actor.latestSnapshot = snapshot ref
      → 清除Worker assignment和physical location
  → controller将snapshot name复制到ActorTemplate.status.goldenSnapshot
~~~

Suspend使用`ActorTemplate.spec.snapshotsConfig.onCommit`决定content scope；源码见[`workflow_suspend.go:150`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L150)、[`workflow_suspend.go:258`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L258)。

| Scope | Golden Snapshot包含什么 | 普通Actor第一次Resume效果 |
|---|---|---|
| `Full` | process memory、rootfs writable delta、支持snapshot的DurableDir | 可以恢复已经初始化的进程和内存，跳过大部分cold init |
| `Data` | 当前主要是DurableDir内容，不含process memory和普通rootfs delta | 恢复data后仍cold boot application，不能提供完整内存预热 |

所以“golden snapshot等于预热内存镜像”只在Full scope下成立；Data scope更接近共享初始持久数据。

若配置Data scope却没有任何DurableDir，gVisor和microVM checkpoint当前都会返回FailedPrecondition；Data不是“空内存snapshot”。证据：[`cmd/ateom-gvisor/main.go:374`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L374)、[`cmd/ateom-microvm/checkpoint.go:75`](../../agent-substrate/substrate/cmd/ateom-microvm/checkpoint.go#L75)。

### 4.7 普通Actor如何使用Golden Snapshot

普通Actor第一次Create只生成SUSPENDED logical record；`CreateActor`不会主动把Template的golden snapshot ref复制进Actor。只有调用者显式传`source_snapshot`时，它才会被canonicalize并写成Actor的`LatestSnapshot`。Resume时，ateapi按以下优先级选择source：

~~~text
1. Actor.latestSnapshot存在
   → 使用Actor自己的最新snapshot

2. Actor没有latestSnapshot
   且ActorTemplate.status.goldenSnapshot非空
   且ResumeActor.boot == false
   → 使用模板共享Golden Snapshot

3. 都不满足
   → 从ActorTemplate spec cold boot
~~~

证据：[`workflow_resume.go:86`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L86)。`boot=true`的proto语义明确是跳过**golden snapshot**；Actor自己的`LatestSnapshot`分支优先于该判断，不应把它解释为“忽略所有snapshot”：[`ateapi.proto:341`](../../agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto#L341)。

这里还有两个需要明确的边界：

- PAUSED Actor若有node-local snapshot，实际restore执行分支会先走local checkpoint；它不改变新Actor首次Resume使用golden的结论。证据：[`workflow_resume.go:461`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L461)。
- `docs/api-guide.md`把`boot`概括为“bypasses snapshots”，比当前实现更宽；源码和proto实际只在golden fallback分支检查`!input.Boot`。因此本文以“own latest > local/durable执行选择；无own latest时boot=false才可用golden”为准。
- `docs/architecture.md`把CreateActor描述成初始化时写入Golden Snapshot ref；当前源码实际在first Resume动态读取`ActorTemplate.status.goldenSnapshot`。显式source snapshot、`boot=true`、或Template尚未Ready/尚无golden时，都可能不从golden派生。

Create/source snapshot证据见[`create_actor.go:37`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L37)、[`create_actor.go:102`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L102)。

完整lineage是：

~~~text
ActorTemplate
  └─ Golden Actor cold boot
       └─ Suspend → Golden Snapshot（共享base）
            ├─ Actor A first Resume
            │    └─ A运行并Suspend → Actor A Latest Snapshot
            │         └─ A subsequent Resume优先恢复自己的snapshot
            ├─ Actor B first Resume
            └─ Actor C first Resume
~~~

因此Golden Snapshot通常只承担普通Actor的初始状态；一旦Actor产生自己的latest snapshot，后续状态沿各自lineage演进。

### 4.8 为什么要做Golden Snapshot

如果application cold boot包含：

~~~text
Python/runtime启动
  → import依赖
  → framework初始化
  → 加载模型、索引或baseline cache
  → readiness
~~~

每个Actor都重复执行会让初始化成本接近`O(number of Actors)`。Golden Actor把公共初始化执行一次，Full golden snapshot再被许多Actor恢复，使公共初始化成本更接近`O(number of ActorTemplates)`，代价转化为snapshot download、restore和post-restore page faults。

这种复用也解释了为什么ActorTemplate spec和image digest被设计成immutable/pinned：snapshot必须与container image、sandbox class及runtime assets兼容。

### 4.9 当前实现边界与风险

- **没有独立Warm API**：无readyz时20秒只是启发式，过短会拍到未完成初始化的状态，过长会增加模板Ready延迟。
- **Controller只编排**：snapshot正确性和持久化依赖ateapi、atelet、ateom及GCS/S3，不能只监控Kubernetes controller reconcile成功。
- **失败补偿仍有TODO**：源码注明Resume失败可能遗留认为自己仍被Golden Actor占用的Worker；controller也缺少更完整的Suspend conflict恢复。
- **PhaseFailed没有进入路径**：CRD定义了`PhaseFailed`，但当前Reconcile错误直接返回，controller没有把模板持久化为Failed phase。
- **Template deletion cleanup不足**：当前删除分支不清理Golden Actor/snapshot，snapshot GC在其他调研中也属于未完成能力。
- **Ready不是ateapi硬门禁**：CreateActor只检查Template存在和UID兼容性，不检查`status.phase == Ready`；在golden尚未完成时Resume会走cold boot。因此Ready是正常使用门槛和controller状态，不是当前API的强制admission gate。见[`create_actor.go:62`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L62)。
- **Secret继承取决于scope**：Full golden snapshot会把Golden Actor启动时解析的environment保存在进程状态里，后续Actor继承旧值；Data scope恢复会cold boot并重新materialize workload spec，可能读取较新的Secret（仍受30秒cache影响）。Secret变化不会触发controller自动重建golden snapshot。见[`docs/api-guide.md:109`](../../agent-substrate/substrate/docs/api-guide.md#L109)、[`workload_spec.go:34`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workload_spec.go#L34)。
- **Actor-specific identity不能烘焙进共享内存**：gVisor路径使用每Actor bind mount提供`/run/ate/actor-id`，应用必须fresh read，避免从Full golden snapshot继承Golden Actor名字；microVM实现当前明确丢弃该bind mount，是一个known gap，不能把该保证泛化到所有sandbox class。见[`docs/api-guide.md:119`](../../agent-substrate/substrate/docs/api-guide.md#L119)、[`cmd/ateom-microvm/spec.go:86`](../../agent-substrate/substrate/cmd/ateom-microvm/spec.go#L86)。
- **外部连接需要应用语义**：数据库连接、socket、短期credential、clock/timer等是否可安全checkpoint/restore，不能由Golden Snapshot机制自动保证。

相关controller TODO见[`actortemplate_controller.go:118`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L118)、[`actortemplate_controller.go:152`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L152)。

### 4.10 理解这段代码需要补哪些Kubernetes知识

如果此前主要熟悉container/runtime而不熟悉Kubernetes control plane，建议补，但不需要先系统学习整个Kubernetes。理解ActorTemplate controller所需的最小知识集是：

1. **API object三部分**：`metadata`表示identity/generation，`spec`表示desired state，`status`表示controller观察和推进到的状态；
2. **CRD**：第三方把`ActorTemplate`、`WorkerPool`等新资源类型加入Kubernetes API；
3. **List/Watch与Informer cache**：controller通常从本地cache观察对象变化，而不是轮询etcd；
4. **Reconcile模型**：输入通常只有namespace/name；每次重新读取current state，执行一小步，要求幂等并可重试；
5. **controller-runtime Manager/Reconciler**：Manager负责cache、client、queue和worker goroutine，业务代码实现`Reconcile`；
6. **eventual consistency**：一次Reconcile不保证完成，error、watch event或`RequeueAfter`会驱动后续尝试；
7. **Status Conditions**：向用户表达Ready/Failed等observed state；
8. **Finalizer与owner reference**：删除前清理外部资源、级联回收。当前Golden Actor cleanup缺口正适合用这一知识理解；
9. **组件边界**：API Server保存对象，controller做决策，scheduler放置Pod，kubelet执行Pod。

推荐直接沿源码学习，而不是先阅读所有Kubernetes网络和存储章节：

~~~text
1. ActorTemplate Go type
   pkg/api/v1alpha1/actortemplate_types.go

2. atecontroller如何注册ActorTemplateReconciler
   cmd/atecontroller/main.go

3. SetupWithManager如何watch ActorTemplate
   actortemplate_controller.go:190

4. Reconcile如何按status.phase推进
   actortemplate_controller.go:65

5. 对比WorkerPool controller
   看它如何从WorkerPool desired state生成Deployment
~~~

理解到这里，已经足够读懂Agent Substrate为什么把低频ActorTemplate/WorkerPool留在Kubernetes，而把高频Actor assignment放进ateapi/Redis。CRI、CNI、CSI、Service网络等知识对理解完整系统仍有帮助，但不是读懂Golden Actor controller的前置条件。

参考入口：[Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)、[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)、[Kubebuilder Controller Implementation](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation)。

## 5. ateapi：Actor层的中心控制面

### 5.1 状态与API

ateapi不是Kubernetes API Server的别名。它提供Agent Substrate自己的gRPC API，并把高频记录放在Redis/ValKey：

- Actor lifecycle status；
- Actor UID/version和snapshot reference；
- Worker Pod、Pod UID、IP、node和sandbox class；
- Worker assignment；
- lifecycle lock和optimistic CAS。

Kubernetes仍保存ActorTemplate、WorkerPool、SandboxConfig和Worker Pods；Actor与Worker assignment不是Kubernetes Pod/CRD lifecycle。

### 5.2 Actor scheduler

Resume时，ateapi从进程内Worker cache取得候选Worker，过滤sandbox class、selector、local snapshot locality等约束。最关键的capacity条件是：

~~~go
worker.GetAssignment() == nil
~~~

只有assignment为空的Worker才能成为候选。没有空闲Worker时返回`ErrNoCapacity`，而不是让同一个Worker追加第二个Actor assignment。证据：[`scheduling.go:89`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go#L89)。

### 5.3 Assignment record的基数

Worker protobuf中是一个singular字段：

~~~protobuf
message Worker {
  ...
  Assignment assignment = 4;
  ...
}

message Assignment {
  KubeNamespacedObjectRef actor_template = 1;
  ObjectRef actor = 2;
}
~~~

不是`repeated Assignment`，也没有slot array。证据：[`ateapi.proto:432`](../../agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto#L432)。

### 5.4 Resume中的claim

ateapi先在Worker cache查找既有assignment或挑选空闲Worker，再把`assignedWorker.Assignment`设为当前Actor，随后把Actor改为RESUMING并写入：

- ateom Pod namespace/name/IP/UID；
- WorkerPool name。

证据：[`workflow_resume.go:222`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L222)、[`workflow_resume.go:280`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L280)。

这个模型本身就把Worker定义成exclusive slot。

## 6. atelet：每Node一个、面向多个Worker的node supervisor

### 6.1 为什么atelet不放在每个Worker里面

atelet需要复用node-local能力：

- image layer和runtime binary cache；
- snapshot download/upload；
- OCI bundle生成；
- Actor node-local目录；
- external volume preparation；
- 对本Node多个ateom socket的连接cache。

将它做成DaemonSet意味着重型artifact准备和host操作不需要在每个Worker Pod重复实现。

### 6.2 atelet如何调用正确的ateom

ateapi已经在Actor record中写入目标Worker Pod UID。atelet的Run/Restore request携带`target_ateom_uid`，随后：

~~~text
target Worker Pod UID
  → /var/lib/ateom-gvisor/ateoms/<podUID>/ateom.sock
  → gRPC ateom client
~~~

socket布局见[`ateompath.go:43`](../../agent-substrate/substrate/internal/ateompath/ateompath.go#L43)，dial逻辑见[`cmd/atelet/main.go:804`](../../agent-substrate/substrate/cmd/atelet/main.go#L804)。

这说明一个atelet可以操纵本Node上的多个ateom，但每次request有明确的目标Worker Pod UID。

### 6.3 Run与Restore的数据准备

fresh Run中，atelet：

1. 校验request；
2. 确保对应sandbox runtime assets存在；
3. 清理/创建Actor目录；
4. 准备volume；
5. 创建pause和application OCI bundles；
6. dial目标ateom；
7. 调用`RunWorkload`。

见[`cmd/atelet/main.go:235`](../../agent-substrate/substrate/cmd/atelet/main.go#L235)。

Restore还会并行处理snapshot下载与OCI/runtime assets preparation，最后调用目标ateom执行restore。因此atelet是artifact/data mover和host orchestrator，不是sandbox内application runtime。

## 7. ateom：每个Worker Pod内部的sandbox driver

### 7.1 对外接口是单workload状态机

ateom protobuf注释明确写成控制“a single gVisor guest inside a worker pod”，并定义两个状态：

~~~text
available
  ├─ RunWorkload ───────→ executing
  └─ RestoreWorkload ───→ executing

executing
  └─ CheckpointWorkload → available/free
~~~

证据：[`ateom.proto:21`](../../agent-substrate/substrate/internal/proto/ateompb/ateom.proto#L21)。

这里的workload对应一个Actor activation，可以包含一个或多个container。

该proto注释仍保留“or, in the future microVM”的旧措辞，但同一commit已经包含完整的`cmd/ateom-microvm`实现。因此“future”是滞后的注释，不能据此判断microVM尚未实现；single-guest和available/executing语义仍与当前架构一致。

### 7.2 gVisor ateom

`ateom-gvisor`负责：

- 在Worker Pod内部mount rootfs overlay；
- 创建和启动pause container；
- 创建/启动每个application container；
- 等待readyz；
- 激活atunnel ingress/egress；
- checkpoint根sandbox并删除runtime container；
- 从checkpoint恢复pause和application containers。

源码入口：[`cmd/ateom-gvisor/main.go:259`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L259)、[`main.go:349`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L349)、[`main.go:479`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L479)。

### 7.3 microVM ateom

`ateom-microvm`负责：

- 启动Cloud Hypervisor VMM；
- 连接Kata agent；
- 创建guest sandbox与network；
- 在同一个VM内启动Actor的containers；
- pause/snapshot/teardown VM；
- OnDemand方式恢复VM memory并重新连接route/logging。

源码入口：[`cmd/ateom-microvm/run.go:187`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L187)、[`restore.go:43`](../../agent-substrate/substrate/cmd/ateom-microvm/restore.go#L43)、[`checkpoint.go:40`](../../agent-substrate/substrate/cmd/ateom-microvm/checkpoint.go#L40)。

### 7.4 atunnel为什么也是并发度证据

atunnel是长期存在于Worker中的activation-aware proxy，但只保存一个：

~~~go
active *activation
~~~

`Activate`注释和错误路径明确规定“There can be only one active actor per worker”。如果已有active Actor，再次Activate会失败。证据：[`internal/atunnel/server.go:52`](../../agent-substrate/substrate/internal/atunnel/server.go#L52)、[`server.go:175`](../../agent-substrate/substrate/internal/atunnel/server.go#L175)。

因此request routing层同样不是Actor→sandbox映射表，而是Worker当前唯一Actor的activation slot。

## 8. “Worker Pod不是最小单位”应该怎样准确表达

### 8.1 正确的部分

如果“运行时最小单位”指不可信Agent代码的隔离和checkpoint边界，那么Worker Pod不是最终边界：

- gVisor class：实际Actor process位于runsc sandbox；
- microVM class：实际Actor process位于Cloud Hypervisor guest；
- Actor可以checkpoint后离开该Worker，并在另一个Worker恢复；
- Worker Pod的lifecycle和Actor process lifecycle被ateom刻意解耦。

所以Actor sandbox才是execution/snapshot boundary。

### 8.2 需要保留的限制

如果“最小单位”指Kubernetes capacity或Substrate physical concurrency slot，则Worker Pod仍然是关键单位：

- kube-scheduler调度的是Worker Pod；
- CNI为Worker Pod建立外层Pod IP；
- Worker record一一映射Worker Pod；
- 一个Worker只有一个assignment；
- WorkerPool replica数基本决定同时可运行Actor数量的上限。

Worker Pod的“warm”主要表示Pod placement、ateom进程、Pod network identity以及node-local cache/hostPath已经就绪；Worker处于available时，内部不一定预先运行着一台gVisor sandbox或microVM。内层sandbox通常在Actor Run/Resume时fresh boot或从snapshot restore。

因此更准确的表述是：

> Worker Pod是外层Kubernetes placement与warm capacity unit；其内部的gVisor/microVM是Actor security execution unit。两者处于不同层次，不能简单说其中一个“不是单位”。

## 9. 一个Worker Pod能否同时启动多个Actor sandbox

### 9.1 受支持架构语义：并发度为1

至少有四层证据共同给出`capacity = 1`：

| 层 | 证据 |
|---|---|
| 文档模型 | Worker “hosts at most one Actor at a time” |
| 数据模型 | `Worker.assignment`是singular field |
| scheduler | 只选`assignment == nil`的Worker |
| request routing | atunnel只有一个`active *activation`并拒绝第二次Activate |
| ateom协议 | available/executing单workload状态机 |

因此不能把Worker Pod理解为一个可同时装入许多独立Actor sandbox的通用node agent。

### 9.2 时间复用：一个Worker先后服务多个Actor

~~~text
t0: Worker available
t1: claim Worker for Actor A
t2: Run/Restore A; Worker executing
t3: Checkpoint A
t4: 清除Worker.assignment; Worker available
t5: claim same Worker for Actor B
t6: Run/Restore B
~~~

这正是Glossary中“many Actors are multiplexed across a pool over time”的含义。设总Actor数为`A`，Worker Pod数为`W`：

~~~text
logical population = A
maximum active Actor concurrency ≈ W
典型目标：W << A，因为大量Actor处于SUSPENDED
~~~

“高密度”主要来自休眠Actor不占Worker和VM/process memory，而不是一个Worker同时运行很多Actor。

### 9.3 microVM中的`running map`是否说明支持多个VM

microVM AteomService包含：

~~~go
running map[string]*runningActor
~~~

见[`cmd/ateom-microvm/main.go:249`](../../agent-substrate/substrate/cmd/ateom-microvm/main.go#L249)。这个map按Actor UID保存live VMM handle，便于Checkpoint找到对应Cloud Hypervisor process和API socket。

不能只因为它是map就推出“一个Worker正式支持多个并发microVM”，原因是：

- 上游Worker record只有一个assignment；
- scheduler不会给已占用Worker第二个Actor；
- ateom lifecycle RPC用mutex串行；
- Worker只有一个per-activation interior netns/network setup；
- atunnel只有一个active Actor。

但从defensive enforcement看，当前ateom实现没有把全部exclusive-slot invariant封装成一个显式`if executing { reject }`状态机。`RunWorkload`会先deactivate现有Actor network，再启动新workload；microVM map可以暂时保留多个UID entry。正常ateapi path不会这样调用，但如果控制面bug或手工错误调用绕过assignment，节点层未必在最早位置拒绝第二个sandbox，可能产生旧VM仍存活但不可路由的orphan。

所以最准确的源码判断是：

> **公开调度模型和受支持语义是一个Worker同时一个Actor；该不变量主要由ateapi assignment和atunnel维护，ateom节点层的硬性防御仍可加强。**

建议增加显式`available/executing`状态、current Actor UID/generation校验，以及Run/Restore时的`FailedPrecondition`，使协议注释成为节点端可验证invariant。

## 10. 一个Actor内部的多个container为什么不是多个sandbox

### 10.1 API模型

ateom的`WorkloadSpec`是：

~~~protobuf
message WorkloadSpec {
  repeated Container containers = 1;
}
~~~

证据：[`ateom.proto:75`](../../agent-substrate/substrate/internal/proto/ateompb/ateom.proto#L75)。所以一个ActorTemplate可以描述多个application containers。

当前ActorTemplate API还对用户声明的application containers设置了最多10个的校验上限；microVM实现内部的25只是更宽的defensive sanity cap，不代表公开API允许25个。证据：[`actortemplate_types.go:313`](../../agent-substrate/substrate/pkg/api/v1alpha1/actortemplate_types.go#L313)、[`cmd/ateom-microvm/run.go:108`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L108)。

### 10.2 gVisor：一个pause sandbox，多个application containers

atelet为pause bundle写入：

~~~text
io.kubernetes.cri.container-type = sandbox
io.kubernetes.cri.container-name = pause
~~~

为每个application container写入：

~~~text
io.kubernetes.cri.container-type = container
io.kubernetes.cri.sandbox-id = pause
~~~

证据：[`cmd/atelet/main.go:727`](../../agent-substrate/substrate/cmd/atelet/main.go#L727)、[`cmd/atelet/main.go:768`](../../agent-substrate/substrate/cmd/atelet/main.go#L768)。

ateom随后先`runsc create/start pause`，再循环create/start所有application containers：[`cmd/ateom-gvisor/main.go:302`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L302)。Checkpoint也只对sandbox root/pause做核心checkpoint，再按container恢复。因此这些container共享同一个Actor gVisor sandbox。

### 10.3 microVM：一个VM，多个guest containers

microVM实现明确写道，Actor的containers“all share the one micro-VM + virtiofsd”，并设置每Actor最多25个container的sanity cap：[`cmd/ateom-microvm/run.go:108`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L108)。

启动时只做一次：

- guest sandbox creation；
- guest network configuration；

然后循环启动每个container。证据：[`cmd/ateom-microvm/run.go:617`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L617)。

因此：

~~~text
错误理解：
  1 Worker Pod → 1 ateom → N Actor microVMs

当前模型：
  1 Worker Pod → 1 ateom → 1 Actor microVM → N Actor containers
~~~

## 11. gVisor与microVM层级对照

| 层 | gVisor Worker | microVM Worker |
|---|---|---|
| 外层Kubernetes对象 | 一个Worker Pod | 一个Worker Pod |
| Pod内driver | `ateom-gvisor` | `ateom-microvm` |
| active Actor隔离边界 | 一个runsc sandbox | 一个Cloud Hypervisor VM |
| 多container实现 | pause sandbox + application containers | 一个guest sandbox内多个Kata containers |
| snapshot | runsc checkpoint/fscheckpoint | CH pause + sparse memory snapshot + durable volume tar |
| resume | runsc restore | CH OnDemand restore + resume |
| 同时active Actors/Worker | 1 | 1（受支持模型） |
| Worker复用 | checkpoint后换下一个Actor | teardown VM后换下一个Actor |

WorkerPool通过`sandboxClass`决定某批Worker使用哪种ateom image。一个Worker Pod不是运行中在gVisor和microVM之间动态切换；不同sandbox class对应不同Worker capacity pool和runtime assets。

## 12. Resume与Suspend如何完成slot复用

### 12.1 Resume

~~~text
1. ateapi锁Actor并读取snapshot/template
2. 从Worker cache选择assignment为空、sandbox class匹配的Worker
3. CAS写Worker.assignment = Actor
4. CAS写Actor = RESUMING + Worker Pod/UID/IP
5. 调目标Node atelet
6. atelet准备snapshot、OCI bundle、runtime assets
7. atelet按Pod UID dial ateom UDS
8. ateom Run/Restore一个Actor workload
9. 等待所有readyz
10. atunnel Activate当前Actor
11. Actor变为RUNNING
~~~

### 12.2 Suspend

~~~text
1. Actor变为SUSPENDING
2. atelet调用目标ateom CheckpointWorkload
3. ateom停止入口、checkpoint并清理runtime
4. atelet上传snapshot/manifest
5. 更新Actor snapshot reference
6. 清除Worker.assignment
7. 清除Actor physical Worker location
8. Actor变为SUSPENDED
9. Worker重新进入free candidates
~~~

Worker释放源码见[`workflow_suspend.go:216`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L216)。

## 13. 这种一Worker一Actor设计的取舍

### 13.1 优点

- capacity accounting简单：Worker是否空闲由一个assignment判断；
- network routing简单：一个Worker IP对应当前唯一Actor；
- atunnel只需保存一个activation；
- snapshot/cleanup可以重置整个slot；
- 避免同一个Worker内多个不可信Actor之间的资源干扰和stale route；
- Kubernetes Pod request/limit可近似代表一个active Actor slot的上限。

### 13.2 成本

- 最大并发Actor数量不超过ready Worker Pod数量；
- 每个并发Actor需要一个外层Worker Pod和一个ateom；
- Worker池扩大仍走Kubernetes Deployment/HPA/Pod启动路径；
- 轻量Actor不能共享一个Worker Pod剩余CPU/memory做细粒度bin packing；
- 不同Actor规格需要不同WorkerPool或接受slot内部资源浪费；
- idle Worker虽无Actor sandbox，仍保留Pod、ateom、network和基础内存。

### 13.3 如果未来要支持一个Worker并发多个Actor

这不是把`running`从pointer改成map就够了，至少需要重新设计：

- `Worker.assignment`改为多slot/多assignment和per-slot generation；
- scheduler做Worker内部CPU/memory/device bin packing；
- ateom强制管理多个独立sandbox lifecycle；
- 每Actor独立netns/veth/IP和atunnel routing table；
- per-Actor cgroup、quota和OOM/failure isolation；
- UDS、snapshot目录、runtime assets和cleanup的并发安全；
- Actor A checkpoint时不能影响Actor B的network、mount或runtime；
- Worker drain与Pod failure需要一次恢复多个Actor；
- request routing必须从“Worker当前Actor”改为“Worker上的Actor map”。

这会提高slot利用率，但也把Worker Pod变成一个mini-node，ateom接近完整node runtime。当前一Worker一Actor是在实现复杂度、隔离和密度之间选择更简单的点。

## 14. 常见误解与准确表述

| 容易误解的说法 | 准确表述 |
|---|---|
| “Actor就是Worker Pod” | Actor是logical lifecycle/snapshot unit；Worker Pod是可复用physical slot |
| “atelet在Worker Pod里面” | atelet是每Node DaemonSet；ateom在Worker Pod里面 |
| “ateom负责调度Actor” | ateapi负责选择Worker；ateom只执行指定Actor的Run/Restore/Checkpoint |
| “Worker Pod内有多个Actor sandbox” | 当前同时最多一个；不同Actor按时间复用同一Worker |
| “一个Actor有多个container，所以有多个sandbox” | gVisor containers共享pause sandbox；microVM containers共享同一个VM |
| “high density表示一个Pod并发很多Actor” | 主要表示大量SUSPENDED Actors复用较小Worker池，`W << A` |
| “microVM的running map证明多VM并发” | map是handle bookkeeping；assignment、network和route的正式模型仍为单Actor slot |
| “Worker Pod不是运行单位” | 它不是最终安全边界，但仍是Kubernetes placement和Substrate并发capacity unit |
| “Golden Actor对应一个专用Golden Pod” | 当前controller创建普通logical Actor，并把它Resume到已有Worker Pod |
| “Warm是Actor的一种RPC” | 当前没有WarmActor；只有readyz gate或默认20秒等待 |
| “所有Actor创建时都复制golden snapshot ref” | Create只写SUSPENDED record；first Resume在无own snapshot且boot=false时动态选择golden |
| “Golden Snapshot总能恢复预热内存” | 只有Full scope保存process/VM state；Data scope恢复会cold boot |
| “ActorTemplate controller是Kubernetes内置controller” | 它是atecontroller进程中用controller-runtime注册的custom controller，watch自定义ActorTemplate CRD |
| “Kubernetes controller只能创建Pod” | Reconcile可以操作Kubernetes对象，也可以调用ateapi等外部系统；核心是把desired state推进到observed state |

## 15. 源码证据索引

| 判断 | 证据 |
|---|---|
| atecontroller注册ActorTemplate custom Reconciler | [`cmd/atecontroller/main.go:114`](../../agent-substrate/substrate/cmd/atecontroller/main.go#L114)、[`actortemplate_controller.go:190`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L190) |
| WorkerPool controller对比：把WorkerPool reconcile为Deployment | [`workerpool_controller.go:81`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_controller.go#L81) |
| ActorTemplate controller状态机与Golden Actor创建 | [`actortemplate_controller.go:65`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L65)、[`actortemplate_controller.go:80`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L80) |
| Golden Actor Resume、ready/warm-up与Suspend | [`actortemplate_controller.go:118`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L118)、[`actortemplate_controller.go:138`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L138)、[`actortemplate_controller.go:145`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L145) |
| readyz完整时warm-up为0，否则默认20秒 | [`actortemplate_controller.go:35`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L35)、[`actortemplate_controller.go:195`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L195) |
| 普通Actor的own latest、golden、cold boot选择顺序 | [`workflow_resume.go:70`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L70)、[`workflow_resume.go:481`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L481) |
| CreateActor只在显式source snapshot时设置LatestSnapshot | [`create_actor.go:37`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L37)、[`create_actor.go:102`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go#L102) |
| Suspend按OnCommit创建ActorSnapshot metadata | [`workflow_suspend.go:150`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L150)、[`workflow_suspend.go:258`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L258) |
| Data scope无DurableDir会失败 | [`cmd/ateom-gvisor/main.go:374`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L374)、[`cmd/ateom-microvm/checkpoint.go:75`](../../agent-substrate/substrate/cmd/ateom-microvm/checkpoint.go#L75) |
| microVM缺少per-Actor identity bind mount | [`cmd/ateom-microvm/spec.go:86`](../../agent-substrate/substrate/cmd/ateom-microvm/spec.go#L86) |
| Golden与Last Snapshot官方词汇定义 | [`docs/glossary.md:120`](../../agent-substrate/substrate/docs/glossary.md#L120) |
| API Guide中的temporary Golden Pod表述 | [`docs/api-guide.md:236`](../../agent-substrate/substrate/docs/api-guide.md#L236) |
| Worker同一时刻最多一个Actor | [`docs/glossary.md:43`](../../agent-substrate/substrate/docs/glossary.md#L43) |
| ateapi、atelet、ateom官方定义 | [`docs/glossary.md:49`](../../agent-substrate/substrate/docs/glossary.md#L49)、[`glossary.md:56`](../../agent-substrate/substrate/docs/glossary.md#L56)、[`glossary.md:60`](../../agent-substrate/substrate/docs/glossary.md#L60) |
| ateapi state store、scheduler、workflow职责 | [`docs/architecture.md:300`](../../agent-substrate/substrate/docs/architecture.md#L300) |
| atelet/ateom node subsystem和runtime class | [`docs/architecture.md:313`](../../agent-substrate/substrate/docs/architecture.md#L313)、[`architecture.md:325`](../../agent-substrate/substrate/docs/architecture.md#L325) |
| Worker只有singular assignment | [`pkg/proto/ateapipb/ateapi.proto:432`](../../agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto#L432) |
| scheduler只选unassigned Worker | [`cmd/ateapi/internal/scheduling/scheduling.go:89`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go#L89) |
| Resume claim Worker并写Actor physical location | [`workflow_resume.go:222`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L222)、[`workflow_resume.go:280`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L280) |
| Suspend后清除Worker assignment | [`workflow_suspend.go:216`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L216) |
| ateom available/executing协议 | [`internal/proto/ateompb/ateom.proto:21`](../../agent-substrate/substrate/internal/proto/ateompb/ateom.proto#L21) |
| 一个workload允许多个containers | [`ateom.proto:75`](../../agent-substrate/substrate/internal/proto/ateompb/ateom.proto#L75) |
| atelet按Pod UID dial目标ateom | [`cmd/atelet/main.go:804`](../../agent-substrate/substrate/cmd/atelet/main.go#L804)、[`internal/ateompath/ateompath.go:43`](../../agent-substrate/substrate/internal/ateompath/ateompath.go#L43) |
| Worker Pod模板只有一个外层ateom container | [`workerpool_apply.go:58`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go#L58)、[`workerpool_apply.go:138`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go#L138) |
| atelet准备并调用RunWorkload | [`cmd/atelet/main.go:235`](../../agent-substrate/substrate/cmd/atelet/main.go#L235) |
| gVisor pause与application container共享sandbox-id | [`cmd/atelet/main.go:727`](../../agent-substrate/substrate/cmd/atelet/main.go#L727)、[`main.go:768`](../../agent-substrate/substrate/cmd/atelet/main.go#L768) |
| gVisor按一个workload启动多个containers | [`cmd/ateom-gvisor/main.go:302`](../../agent-substrate/substrate/cmd/ateom-gvisor/main.go#L302) |
| atunnel只有一个active Actor | [`internal/atunnel/server.go:52`](../../agent-substrate/substrate/internal/atunnel/server.go#L52)、[`server.go:175`](../../agent-substrate/substrate/internal/atunnel/server.go#L175) |
| microVM containers共享一个VM | [`cmd/ateom-microvm/run.go:108`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L108)、[`run.go:617`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L617) |
| ActorTemplate公开API最多10个application containers | [`actortemplate_types.go:313`](../../agent-substrate/substrate/pkg/api/v1alpha1/actortemplate_types.go#L313) |
| microVM lifecycle mutex和running map | [`cmd/ateom-microvm/main.go:249`](../../agent-substrate/substrate/cmd/ateom-microvm/main.go#L249) |

## 16. 最终结论

Agent Substrate采用的是二层执行模型：Kubernetes预先创建Worker Pods作为warm physical slots，ateom再在这些Pod内部创建或恢复真正的Actor sandbox。ateapi负责logical Actor到physical Worker的exclusive assignment，atelet负责Node上的artifact和runtime准备，ateom负责指定slot内的sandbox生命周期。

当前受支持模型不是：

~~~text
一个Worker Pod同时bin-pack多个Actor sandbox
~~~

而是：

~~~text
一个Worker Pod同一时刻运行一个Actor sandbox，
大量Actor通过Suspend/Resume在Worker池上做时间复用，
一个Actor sandbox内部可以包含多个application containers。
~~~

所以Worker Pod既不是最终的Agent安全隔离边界，也不是可以忽略的外壳；它是Kubernetes placement、预热容量和并发slot单位。gVisor sandbox或microVM才是Actor process、memory和snapshot的执行边界。

在这个二层执行模型之上，ActorTemplate controller再增加一层**模板初始化复用**：Golden Actor只负责把公共cold-start初始化做一次，Golden Snapshot成为许多普通Actor的共享初始checkpoint；普通Actor恢复后再沿各自Last Snapshot演进。这种机制减少的是每Actor重复初始化，而不是让一个Worker同时运行多个Actor。

## 17. 参考资料

1. Agent Substrate，[Glossary](../../agent-substrate/substrate/docs/glossary.md)，固定commit见文首。
2. Agent Substrate，[Architecture](../../agent-substrate/substrate/docs/architecture.md)，固定commit见文首。
3. Agent Substrate，[API Guide](../../agent-substrate/substrate/docs/api-guide.md)，其设计说明与当前源码差异见第4节。
4. Agent Substrate protobuf与Go runtime实现，具体路径和行号见第15节。
5. [`Kubernetes、Agent Substrate与AKernel：Control Plane、Data Plane与Kubernetes Bypass源码级架构对比`](./20260804T093336Z-kubernetes-agent-substrate-akernel-control-data-plane-bypass-survey.md)。
6. [`Agent Substrate Kubernetes declarative tax与AKernel Survey`](./20260803T121206Z-agent-substrate-kubernetes-declarative-tax-vs-akernel-survey.md)。
7. Kubernetes官方文档，[Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)与[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)。
8. Kubebuilder Book，[Controller Implementation](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation)。
