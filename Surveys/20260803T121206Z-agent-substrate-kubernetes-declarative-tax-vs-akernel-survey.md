# Kubernetes Declarative Tax、Agent Substrate 专用控制面与 AKernel 源码级对比调研

> 调研时间：2026-08-03 20:12:06（UTC+8）  
> 文档时间戳：20260803T121206Z  
> 核心论文：Qingyuan Liu et al., The Gap Between Serverless Research and Real-world Systems, SoCC 2023  
> 本地 PDF SHA-256：26ec311f00f2f532578750cc3948a3cb613f00cd13b8eb311c7847ac175b2ead  
> Agent Substrate 源码：cbdeb7dbe003a55a16960a301bc595d9aa38b1ad  
> AKernel 源码：5684c858cd835db053adef1265419af64e67bd04  
> sandboxd 子模块：60e6a33e1f2b1ae04c10586c238f6d0200d0c315  
> distill-fs 子模块：5f3d6d3979a4ed76d4bb6722d7c1ce6db7126634  
> Actor 补充审计时间：2026-08-04  
> Ray OSDI 2018 本地 PDF SHA-256：066fecee9604ca232b5fbaeaa7dd260c88149a1be6dd4357ef16705986b99290  
> openYuanRong ray-adapter 补充源码：fb131246f2469b1a6ab1d03d8a3c9cf7d78354d1  
> 网页访问时间：2026-08-03  
> 方法：论文和网页用于定位问题与术语，固定版本源码用于确认当前实现。本次没有在真实集群中运行 benchmark；文中的性能判断分为“源码能够确定的关键路径缩短”“作者或项目报告的数据”“仍需实验验证的假设”。

## 1. 结论摘要

### 1.1 最重要的结论

Agent Substrate 的 “A control plane beside Kubernetes, not inside it” 不是完全绕开 Kubernetes，而是做了一个**时间尺度和对象粒度的分层**：

~~~text
低频、粗粒度、需要治理与持续协调
  -> Kubernetes
  -> WorkerPool / ActorTemplate / SandboxConfig
  -> Deployment / Worker Pod / HPA / node lifecycle

高频、细粒度、面向一次 Actor create/resume/suspend
  -> Agent Substrate 专用控制面
  -> Redis/ValKey Actor + Worker 状态
  -> 内存 Worker cache
  -> versioned worker claim
  -> atelet / ateom direct RPC
  -> sandbox snapshot restore
~~~

Kubernetes 仍然负责准备和维持物理 Worker 容量，但一条 Actor 唤醒请求不再为该 Actor：

- 创建 Pod object；
- 等待 Deployment/ReplicaSet controller；
- 经过 kube-scheduler placement/binding；
- 等待 kubelet syncPod；
- 重新创建 Pod sandbox 和 CNI network；
- 为 Actor 创建 Service/EndpointSlice；
- 通过 Pod status/watch 才知道应用可用。

Substrate 改为从已运行的 Worker Pod 中 claim 一个空闲 slot，再直接调用节点 atelet 和 Worker 内 ateom 恢复 gVisor 或 Cloud Hypervisor sandbox。它把 Kubernetes 的 per-Pod 税移到 WorkerPool 的低频扩缩容和后台预热路径。

### 1.2 为什么传统 Kubernetes 声明式创建在高并发时可能慢

Liu 等在 Declarative Tax 一章中的核心论证是：

1. 调用方修改 desired state；
2. API Server/etcd 持久化并通知 watcher；
3. 多个 controller 各自排队、观察和 reconcile；
4. controller 再通过 API Server 创建或更新下一层对象；
5. 最终 Pod 才进入 scheduling 和 node preparation。

OpenFaaS 案例从 Deployment replicas 由 1 改为 2，到 Pod 创建，Figure 4 给出至少六次必要的 API Server 消息。高并发不是简单把一次固定延迟乘以并发数；更严重的是 API Server、etcd、controller workqueue、scheduler、kubelet、runtime、CNI 和镜像存储成为共享排队点，使 tail latency 和方差随 burst 增长。

但必须保留两个边界：

- 论文第 3 章没有受控端到端 Declarative Tax 实验；
- Figure 4 的至少六条消息不是所有 Kubernetes Pod 创建的固定消息数，也没有证明所有消息严格串行。

因此最稳妥的表述是“声明式多组件收敛引入额外消息、排队和不确定性”，不能直接把它写成某个固定毫秒值。

### 1.3 Substrate 并没有消除状态同步

Substrate 把通用 Kubernetes 同步链替换为较窄的专用同步链：

~~~text
router ResumeActor
  -> Actor distributed lock
  -> Redis Get Actor
  -> local Worker cache scan
  -> Redis CAS claim Worker
  -> Redis update Actor = RESUMING
  -> direct atelet Restore
  -> Redis update Actor = RUNNING
~~~

所以它优化的是：

- 对象数量；
- controller 数量；
- API 层级；
- placement 范围；
- runtime 准备粒度；
- 热路径可观测性与同步性。

它没有获得“零协调”。Worker 和 Actor 更新还不是跨记录原子事务，源码需要在重试时扫描旧 assignment 做前向恢复；Redis、对象存储、snapshot restore、OCI unpack 和 readiness 都可能成为新的瓶颈。

### 1.4 AKernel 与 Substrate 的共同方向

AKernel 当前也属于 Kubernetes-hosted、beside-per-Pod-Kubernetes 的二级 execution substrate：

- Kubernetes 部署长期 master、frontend、node DaemonSet；
- 每个 Sandbox 不对应一个 Kubernetes Pod；
- SDK 通过 openYuanRong 创建 instance；
- global scheduler 选择 node；
- node 内 openYuanRong runtime 通过 Unix socket 连接 sandboxd；
- sandboxd 直接准备 rootfs、cgroup、veth/IP 并调用 runsc。

因此 AKernel 同样避开了 per-Sandbox 的 API Server/etcd、kube-scheduler、kubelet/CRI/CNI/CSI Pod 链。

### 1.5 AKernel 与 Substrate 的根本差异

| 维度 | Agent Substrate | AKernel 当前源码 |
|---|---|---|
| 复用粒度 | 预热完整 Worker Pod slot；在其内部 restore Actor sandbox | cgroup、veth/IP、rootfs mount/cache 等 host resource |
| 逻辑/物理解耦 | Actor 与 Worker 分离，可重新映射 | Sandbox handle 最终对应 physical sandbox ID |
| 创建语义 | CreateActor 只写 SUSPENDED 逻辑记录，首次请求再 Resume | Sandbox constructor 触发实际 instance 创建并等待 actor ping |
| 节点启动 | snapshot restore 或从模板 boot | 每个 Sandbox fresh runsc create |
| idle 状态 | 显式 suspend/pause，snapshot 后释放 Worker | idle_timeout 可回收，但没有公开 execution checkpoint/restore |
| 控制状态 | Actor/Worker Redis state + version + lock/workflow | openYuanRong 控制面是预编译依赖；sandboxd 有节点局部 metadata/store |
| 饱和处理 | bounded request parking + singleflight | 有 schedule_timeout、资源池 max/fail-fast；没有等价 Actor request parking |
| per-instance resources | 主要由 WorkerPool fixed shape 决定 | SDK 有 per-Sandbox CPU/memory request/limit |
| 可审计程度 | scheduler/workflow/store 主要源码可见 | openYuanRong scheduler、queue 和一致性逻辑不在本仓库 |

Substrate 更接近“把大量 idle logical Actor multiplex 到少量 active Worker”；AKernel 当前更接近“为每个 Sandbox 提供专用调度和更短的 fresh gVisor creation fast path”。

### 1.6 两边都叫 Actor，但不是同一层抽象

AKernel/openYuanRong Actor 与 Agent Substrate Actor 有共同的 Actor 家族特征：有身份、有私有可变状态、通过 handle 或地址交互，并由系统放置到远端执行资源。但两者不应直接等同：

- Ray OSDI 2018 Actor 是应用级 stateful process/class instance，核心语义是远程方法、future、单 Actor 方法串行和 lineage replay；
- AKernel 当前用 openYuanRong `@yr.instance` 实现 sandbox 内的 `_SandboxInstance` 管理对象，它主要暴露 command/filesystem/tunnel RPC，AI Agent 用户进程反而是这个管理对象启动的子进程；
- Agent Substrate Actor 是完整 agent-like workload/sandbox 的逻辑生命周期身份，通过 HTTP traffic 使用；它没有 Ray 式任意 remote method API，也没有源码级的单 Actor application request 串行保证；
- Substrate Actor 可在 suspend 后释放 Worker，再从 sandbox snapshot 恢复到另一 Worker；AKernel 当前 instance 与 physical sandbox routing identity 绑定，尚无这一层 logical Actor/Worker multiplexing。

因此更准确的结论是：openYuanRong 的 stateful instance 明显是 Ray-like remote actor abstraction，且官方 `ray-adapter` 确实把 Ray `ActorClass/ActorHandle` 映射到原生 `InstanceCreator/InstanceProxy`；但 AKernel 直接使用原生 `yr` 而不是 `ray_adapter`，当前源码不能证明其历史来源就是 Ray。Agent Substrate 则借用了 Actor 的“长期逻辑身份”含义，把抽象边界扩大到了完整 sandbox。第 9 节给出源码级展开。

## 2. 证据范围和成熟度

### 2.1 证据分级

| 类型 | 本文如何使用 |
|---|---|
| 固定版本源码 | 判断 API、关键路径、状态存储、并发、池化和边界 |
| 项目 architecture/README | 判断设计意图；若与源码不同，以源码为准 |
| Liu 等 SoCC 2023 | 解释 Declarative Tax 的问题定义、案例和研究建议 |
| Moritz 等 OSDI 2018 | 固定 Ray Actor 的原始定义、顺序、lineage、placement和故障恢复边界 |
| The New Stack 网页 | 作为二级解释材料，不作为独立性能证据 |
| openYuanRong官方文档与固定ray-adapter源码 | 判断native stateful instance与Ray API的兼容映射；不替代AKernel打包版本的runtime源码 |
| 两份 AKernel 既有 Survey | 复用 Kubernetes Pod 管线和 AKernel handle/resource pool 的详细背景，并重新核对关键源码 |

### 2.2 Agent Substrate 的性能数字应如何解读

Substrate README 报告 demo 将约 250 个 stateful Actor multiplex 到 8 个 Worker Pod，约等于 31.25 个 Actor/Pod，并称 30x+ oversubscription、sub-second activation。

architecture 还列出 north-star targets：

- activation latency：p95 100 ms；
- scale：单集群 10 亿 Actor；
- throughput：每秒 1000 次 wakeup。

这些目标和 demo 不应混为已发表 benchmark：

- architecture 第 3 行明确说大量内容仍是 aspirational；
- README 第 50–53 行明确说项目 early development、not ready for production；
- benchmarking README 把当前工具称为 nascent suite；
- 仓库没有一张在固定硬件、软件和置信区间下可直接引用的正式结果表。

本文只把它们用于说明设计目标。

### 2.3 The New Stack 网页的证据地位

网页中的两个标题准确概括了架构思想：

- Why the Kubernetes control plane sits in the wrong place；
- A control plane beside Kubernetes, not inside it。

网页还复述 30x+ oversubscription 和 sub-second activation。它是第三方解释文章，其数字来自项目报告；本文以 Substrate 源码和 README 为一手证据，不用网页替代源码。

## 3. Liu 等论文的 Declarative Tax

### 3.1 章节边界

本地 PDF 共 11 个物理页。Challenge II 位于：

- PDF 物理页 3 / 论文印刷页 3 右栏开始；
- PDF 物理页 4 / 论文印刷页 4 的 Insight II 结束。

紧随其后的 2,000 concurrent Pods、约 14.5 秒 scheduling cost、1,000→10,000 Pod 对应 1→10 GB scheduler memory，属于 Challenge III: Scheduling Cost，不是 Declarative Tax 本章的实验数据。本文不会混用。

### 3.2 声明式方法的收益

Kubernetes 的调用方声明 expected state，controller 负责让 observed state 最终收敛。它的价值包括：

- 接口简单；
- controller 自动补偿失败；
- 对象可持久化和 watch；
- 易于组合 Deployment、ReplicaSet、Pod、Service、Volume；
- 天然支持 autoscaling、rollout 和 reconciliation；
- 各组件可模块化演进。

论文的目标不是把声明式接口全部换成同步 imperative RPC，而是在保留易用性和模块化的同时降低控制链成本。

### 3.3 OpenFaaS Figure 4 路径

论文案例：

~~~text
OpenFaaS controller
  -> 将 Function-A Deployment replica 从 1 改成 2
  -> API Server / etcd
  -> Deployment controller sync
  -> 更新 ReplicaSet
  -> API Server / etcd
  -> ReplicaSet controller sync
  -> 创建 Pod object
  -> API Server / etcd
  -> Pod scheduling / node start
  -> Instance A-2
~~~

Figure 4(b) 标出至少六条 essential messages through API Server。Knative 等系统再增加 CRD 和 controller 后，round trip 会更复杂。

### 3.4 本章数字和证据属性

| 数字 | 页码 | 含义 | 证据边界 |
|---|---:|---|---|
| replicas 1→2 | p4 Figure 4 | OpenFaaS scale-out 示例 | 架构案例，不是实验参数 |
| 每次 API Server operation 几毫秒 | p4 | 单操作延迟 | 引用外部 OpenStack performance docs，不是本文实测 |
| 至少 6 条 API Server 消息 | p4 | Figure 4 essential sync 下界 | 结构性计数，不是 packet trace |
| runtime initialization 亚毫秒 | p4 | Catalyzer/Faasm 研究水平 | 来自外部论文，未在同一实验环境复现 |
| microsecond-scale latency | p4 | 声明式路径难以支持的目标量级 | 目标判断，不是测量结果 |

本章没有报告：

- 机器、节点和 Kubernetes 版本；
- OpenFaaS/Knative/etcd 拓扑；
- 并发、样本数和重复轮次；
- p50/p95/p99；
- API Server、etcd 和 controller queue 分项时延；
- direct path 与 declarative path 的 ablation。

因此不能根据“几毫秒 × 至少六条消息”自行计算固定 Declarative Tax。消息可能有并行和流水化，外部引用也不在相同环境。

### 3.5 为什么高并发放大这项税

设一次扩容会产生多个对象更新和 controller event。M 个并发 instance 不只产生 M 个 runtime start：

~~~text
M 个 desired-state changes
  -> O(M) API object writes / status updates
  -> O(M) watch events
  -> 多个 controller workqueue
  -> O(M) Pod scheduling/binding
  -> O(M) kubelet/runtime/network/storage preparation
~~~

共享点包括：

- API Server request handling、serialization、authn/z、admission；
- etcd WAL、MVCC、watch fanout；
- Deployment/ReplicaSet/custom controller workqueue；
- scheduler cache、filter/score/bind；
- kubelet sync workers；
- CRI runtime、CNI、CSI、image registry 和 node disk。

不同 Pod 之间可以并行，CRI/CNI/CSI 也不是一个固定的完全串行三函数链；但单 Pod 必须跨过 rootfs/image、required volume、sandbox/network 和 container/application ready 等依赖屏障。burst 会让共享队列积压，增加 tail 和 jitter。

### 3.6 论文给出的方向

单组件：

- 加速 API Server/etcd synchronization；
- 优化 controller queue；
- 降低 controller 带来的时延方差。

跨组件：

- 为 state synchronization 增加 fast path；
- 在保留模块化的情况下联合优化多个组件；
- 探索 hardware-software co-design。

Substrate 正是“保留 Kubernetes 治理面，增加专用高频 fast path”的一种实现。

## 4. 传统 Kubernetes 的 per-Pod 关键路径

### 4.1 Deployment/Job 路径

典型链：

~~~text
client / autoscaler
  -> API Server
  -> authn / RBAC / admission / quota
  -> etcd
  -> Deployment or Job controller
  -> ReplicaSet or Pod object
  -> API Server / etcd
  -> kube-scheduler
  -> Pod binding
  -> kubelet
  -> volume manager / CSI
  -> CRI PodSandbox
  -> CNI
  -> image / snapshot
  -> container create/start
  -> readiness
  -> status / EndpointSlice / routing
~~~

Declarative Tax 主要描述前半段 desired-state convergence；完整 Pod start 还包含 scheduling 和 node data plane。两者不能混成同一个固定数值。

### 4.2 direct Pod create 路径

直接 POST Pod 可以跳过 Deployment/ReplicaSet controller，因此不会固定拥有论文 Figure 4 的全部六条消息，但仍需：

- API Server/etcd；
- admission/quota；
- scheduler/binding；
- kubelet status convergence；
- PodSandbox/network/image/container readiness。

所以“至少六条”是 OpenFaaS Deployment 案例，不是 Kubernetes ABI。

### 4.3 warm Pod + exec 路径

如果先保留一个 Ready Pod，再用 pods/exec 或 Pod 内 agent daemon 启动进程，steady-state command 本来就不再创建 Pod，也不走 scheduler/CNI/image pull。这说明 Substrate 的核心思想不是“只有它能绕过 K8s”，而是：

- 把 warm capacity、logical identity、snapshot、routing、assignment 和 backpressure 组合成系统级服务；
- 让大量 Actor 共享少量 warm Pod；
- 为 suspend/resume 而非诊断 exec 定义一等状态机。

## 5. Agent Substrate 如何将 Kubernetes 移出高频路径

### 5.1 对象数量从 Actor 数降到 active capacity 数

设：

- A：逻辑 Actor 数；
- W：Worker Pod 数；
- T：ActorTemplate/WorkerPool 等配置数；
- A 远大于 W。

传统 per-Actor Pod 的 Kubernetes 对象规模至少与 A 同阶；Substrate 中：

~~~text
Kubernetes objects ≈ O(W + T + control-plane components)
Substrate dynamic records ≈ O(A + W)
~~~

高频 Actor 状态进入 Redis/ValKey，而 Kubernetes 只看到 Worker 容量和模板。这把数百万 Actor 的状态更新从 API Server/etcd 主控制面移出。

### 5.2 低频 CRD 与高频数据库分层

Kubernetes CRD：

- WorkerPool：replicas、Pod resources、selector/affinity、sandbox class；
- ActorTemplate：OCI image、command、env、volumes、readiness、snapshot policy；
- SandboxConfig：runsc 或 microVM assets。

Redis/ValKey：

- Actor：SUSPENDED/RESUMING/RUNNING/SUSPENDING/PAUSED/CRASHED、Worker/IP、snapshot、version；
- Worker：Pod、node、IP、sandbox class、ACTIVE/DRAINING、assignment、version；
- snapshot/tag/Atespace；
- per-Actor distributed lock。

WorkerPool controller 仍把 WorkerPool reconcile 成 Kubernetes Deployment，Deployment replicas 等于 WorkerPool replicas。Kubernetes 的声明式机制负责长期 Worker 的可用性，不负责每次 Actor wake。

### 5.3 CreateActor 的真实语义

源码路径：

1. 校验 Actor；
2. 通过 ActorTemplate informer lister 读取模板；
3. 在 Redis 检查 Atespace；
4. 构造 STATUS_SUSPENDED Actor；
5. Redis SETNX 写入 Actor record；
6. 返回。

CreateActor 不创建 Pod，也不启动 sandbox。

这解释了为什么它可以很轻，但也说明不能把 CreateActor RPC latency 与 Kubernetes Pod application-ready latency直接比较。公平终点应是：

~~~text
CreateActor + first ResumeActor + route + first useful response
~~~

源码还有一个文档边界：普通 CreateActor 没有立即把 golden snapshot ref 写入 Actor latest_snapshot；首次 Resume 的 LoadActorForResumeStep 会从 ActorTemplate status 解析 golden snapshot。模板尚无 golden snapshot 时可以 cold boot。

### 5.4 Worker cache 与专用 scheduler

ateapi 通过 Redis Pub/Sub 和周期 relist 维护进程内 Worker map：

- 启动先同步 Worker；
- 后台订阅 create/update/delete；
- Redis Pub/Sub 断开后 resync；
- 定期全量 relist 修复漏事件；
- 版本较旧的事件不覆盖新记录。

Resume 的 Worker 选择：

1. 复制进程内 Worker slice；
2. 过滤已有 assignment 的 Worker；
3. 要求 STATE_ACTIVE；
4. 匹配 sandboxClass；
5. 匹配 template/actor selector；
6. local pause 时匹配 required node；
7. 从候选中随机选择。

这绕过 kube-scheduler 的通用 filter/score/bind 和 Pod binding，但当前 scheduler 不是复杂 resource-aware bin packing。Worker shape 已经由 WorkerPool Pod 决定。

### 5.5 Worker claim 与一致性

AssignWorker：

1. 先检查以前失败请求是否留下本 Actor assignment；
2. 在 Worker clone 写 Actor assignment；
3. 用 Worker expectedVersion 做 Redis WATCH/CAS；
4. 将 Actor 改为 RESUMING并写 Pod/IP/UID；
5. 用 Actor metadata version 更新 Redis。

两个 Actor 抢同一 Worker 时，Worker 单 key CAS 保证一个成功。

但 Worker 与 Actor 是两个独立事务：

~~~text
UpdateWorker success
  -> process crash
  -> UpdateActor not executed
~~~

下一次 Resume 会扫描 Worker assignment 找回此前 claim。这比 Kubernetes 多 controller 收敛路径更短，却仍需要 lock、version、retry、补偿和 periodic resync。

### 5.6 direct atelet/ateom restore

claim 后：

~~~text
ateapi
  -> 根据 Worker Pod / node informer 找 atelet
  -> cached mTLS gRPC connection
  -> atelet Restore
  -> 读取 local/remote snapshot manifest
  -> 并行：
       snapshot download/decompress
       sandbox assets + OCI bundle preparation
  -> Worker hostPath Unix socket
  -> ateom RestoreWorkload
  -> runsc create/restore 或 CH restore
  -> readyz
  -> Actor RUNNING
~~~

变化的是 Worker Pod 内部承载的 sandbox，不是 Kubernetes Pod 本身。因而没有 per-Actor Pod create/schedule/sync。

### 5.7 Golden Snapshot 把初始化移出请求路径

ActorTemplate controller：

1. 创建 reserved Atespace 的 golden Actor；
2. 用普通 ResumeActor cold boot；
3. 等 readiness；无完整 readyz 时默认等待约 20 秒；
4. SuspendActor；
5. 把 snapshot ID 写入 ActorTemplate status。

首次用户 Actor 可从该快照恢复，避免重复执行应用初始化。这是 latency shifting/amortization，不是初始化成本消失。

### 5.8 routing 不创建 Kubernetes Service

请求到 Envoy：

1. ext_proc 从 Host 解析 Actor；
2. 进入有界 parking lot；
3. 调 ResumeActor；
4. 得到 Worker IP；
5. 将 Envoy original destination 设为 WorkerIP:443；
6. mTLS 连接 Worker atunnel；
7. atunnel 转发到 Actor 私有 veth。

Actor 不需要自己的 Service、Endpoint/EndpointSlice 或 Ingress object。location change 只更新专用 Actor record 和路由结果。

### 5.9 singleflight 和 parking

默认参数：

| 参数 | 默认值 |
|---|---:|
| parking budget | 5 秒 |
| max parked requests | 1024 |
| initial retry interval | 100 ms |
| factor | 1.1 |
| jitter | 0.1 |

同一 Actor 的并发请求由 singleflight 合并为一个 Resume RPC。Worker 暂时不足或控制面短暂不可用时，request 在预算内重试；parking lot 满后立即 shed，避免无界队列。

Parking 不会同步创建 Pod。它主要等待已有 Worker 被释放，或等外部异步 WorkerPool 扩容碰巧完成。

## 6. 为什么这条路径理论上更快

### 6.1 延迟分解

传统 per-Pod：

~~~text
T_k8s_pod
≈ T_api/admission/etcd
 + T_controller convergence
 + T_scheduler/bind
 + T_kubelet/CRI/CNI/CSI/image
 + T_application_ready
~~~

Substrate resume：

~~~text
T_substrate_resume
≈ T_router
 + T_lock/Actor read
 + T_worker-cache scan
 + T_Redis claim/state
 + max(T_snapshot_download, T_assets+OCI_prepare)
 + T_sandbox_restore
 + T_ready
 + T_route
~~~

二式不是在同一抽象层逐项等价，但它清楚显示 Substrate 删除了哪些通用 Pod 阶段。

### 6.2 被消除、被摊销和仍然存在的成本

| 成本 | Substrate 处理 |
|---|---|
| per-Actor API Server/etcd Pod write | 消除；Actor 写 Redis |
| Deployment/ReplicaSet reconcile | 移到 WorkerPool 低频容量路径 |
| kube-scheduler placement | Worker Pod 预先完成；Actor 只选空闲 Worker |
| kubelet PodSandbox/CNI | Worker Pod 预先完成 |
| image/application initialization | golden snapshot 摊销；OCI bundle/restore仍存在 |
| Service/Endpoint route convergence | 专用 router + atunnel 替代 |
| same-Actor concurrent wakeup | singleflight 去重 |
| transient pool saturation | bounded parking |
| snapshot state transfer | 仍存在，可能主导 |
| Redis state/lock/CAS | 新增专用控制面成本 |
| sandbox restore/readiness | 仍存在 |

### 6.3 高并发下的优势来源

- API object amplification 从每 Actor Pod 降为每 Worker Pod；
- Worker placement 已提前完成；
- assignment 是内存 scan + Redis CAS；
- per-Actor routing 不写 Kubernetes object；
- same-Actor thundering herd 被 singleflight 合并；
- parking 明确提供有界 backpressure；
- restore I/O 与 OCI preparation 并行；
- idle Actor snapshot 后不占 Worker。

这些机制能够从源码确认。它们能否达到某个 p95/p99，仍取决于 Worker 数、Redis、snapshot size、storage/cache、runsc/CH restore 和 readiness。

## 7. “绕过 Kubernetes”必须保留的边界

### 7.1 Resume 仍读取 Kubernetes 元数据

- ActorTemplate、WorkerPool、SandboxConfig 来自 informer/lister cache；
- Worker Pod 和 atelet Pod 定位依赖 Pod informer；
- cache freshness 仍受 Kubernetes List/Watch 影响。

这些通常不是同步 API Server RPC，但仍然是 Kubernetes 依赖。

### 7.2 Secret cache miss 是同步 API 调用

ActorTemplate env 包含 secretKeyRef 时，Resume 会解析 workload spec。Secret cache TTL 是 30 秒；miss 时直接调用 Kubernetes Secrets.Get。

因此“热路径完全不访问 API Server”不严格成立。可以说“不创建/调度 per-Actor Pod；多数配置通过 informer cache读取，但 Secret miss 等路径仍可能同步访问 API Server”。

### 7.3 external volume 可能重引入控制路径

Resume workflow 含 CreateVolumes/AttachVolumes。当前全局 external volume plugin 尚不成熟，不能把无 volume 的 fast path 结论直接推广到 CSI-backed stateful workload。

### 7.4 WorkerPool 扩容仍支付 Kubernetes 税

WorkerPool/HPA 扩容链仍是：

~~~text
metric
  -> HPA
  -> WorkerPool /scale
  -> Deployment replicas
  -> new Worker Pod
  -> scheduler / kubelet / runtime
~~~

预热池容量规划正确时，这条链在 actor wake critical path 外；池耗尽时，它会重新暴露为 capacity shortage。5 秒 parking 并不能保证新的 Worker 一定及时 ready。

### 7.5 已运行 Actor 的每个请求仍进控制面

router 对每个 Actor HTTP request 都调用 ResumeActor。Running Actor 的 workflow 会跳过 restore，但仍包含 router gRPC、Actor lock/Get 和模板 cache lookup，并不是建立一次后纯直连的数据面。

### 7.6 当前 scheduler 是 O(W) scan

Worker cache 的读取会复制全部 Worker pointer，scheduler 再线性过滤。AssignWorker 的失败恢复还可能先扫描一遍 assignment。Worker 规模大时，这个新控制面需要按 label/node/free-list 建 index，不能假设永远 O(1)。

### 7.7 Redis Pub/Sub 不是可靠日志

Worker event publish 失败只记录日志；cache 依赖 periodic relist 修复 drift。它比 Kubernetes API object/watch 更轻，但在可靠性、审计和恢复上需要自己补能力。

## 8. AKernel 当前如何避开 per-Sandbox Pod

### 8.1 Kubernetes 只承载长期系统组件

Helm topology：

- master：StatefulSet；
- frontend：Deployment/CloneSet；
- node：每节点一个 privileged DaemonSet；
- etcd、Traefik、monitoring 等长期组件。

node ServiceAccount 的可见 RBAC 只有 Nodes/Pods get/list/watch，用于本机资源视图，没有 per-Sandbox Pod create 权限。这是“Sandbox 不是 Pod”的直接部署证据。

### 8.2 openYuanRong 与 sandboxd 的本地 wiring

node Pod 内 systemd service 设置：

~~~text
CONTAINER_EP=unix:///run/sandboxd/sandboxd.sock
~~~

即 openYuanRong node runtime 通过本地 Unix socket 把 container/sandbox operation 接给 sandboxd。

但需要特别说明：

- openYuanRong 0.9.1 二进制由 Dockerfile 从 OBS tarball 下载；
- Python SDK 依赖 openyuanrong-sdk 0.7.51；
- frontend/master/global scheduler 的核心实现没有 vendored 在 AKernel 仓库。

因此本仓库不能审计：

- scheduler queue；
- placement algorithm；
- fairness/preemption；
- etcd 写放大；
- instance state transaction；
- resource reservation/rollback；
- node failure recovery。

只能把 frontend→master/scheduler→node→sandboxd 标为可观察的系统边界，不能虚构内部算法。

### 8.3 SDK 创建语义

Sandbox 构造：

1. 初始化 openYuanRong client；
2. 组装 InvokeOptions：CPU、memory、limits、idle/schedule timeout、rootfs、mount、port、node affinity；
3. 调 _SandboxInstance.options(...).invoke；
4. 等待远端 actor ping；
5. 返回 Sandbox handle。

返回表示 actor execution context 可 RPC，不等于任意用户 HTTP service 已 ready。它与 Substrate CreateActor 的 SUSPENDED logical record 语义不同。

### 8.4 sandboxd 节点 fast path

进入 sandboxd Start 后：

~~~text
Reserve Sandbox ID
  -> 并行：
       fsMgr.Prepare
       prepareStartResources
          -> 并行：
               Allocate cgroup
               Allocate veth/IP
  -> join
  -> build runtime request / OCI spec
  -> runsc StartSandbox
  -> optional DNAT
  -> FS state commit
  -> return Sandbox ID
~~~

这是一条显式、较窄、节点本地的 imperative path，不需要 Kubernetes controller 为每个 Sandbox 收敛。

### 8.5 cgroup pool

- 启动时填充 idle pool 到 cache size；
- fast path 从 idle queue pop；
- miss 且总量低于 max 时交给维护 goroutine创建；
- 达到 max 立即 ResourceExhausted；
- Sandbox 删除后 recycle 到 idle；
- 周期性 shrink。

命中 pool 省去 cgroup directory/controller 创建，但每个 Running Sandbox 仍独占一个 cgroup，资源限制仍需更新。

### 8.6 veth/IP pool

- 预创建 veth/IP；
- fast allocate 从 idle queue pop；
- recycle 回 idle；
- max 时 fail-fast；
- 创建/销毁由单维护 goroutine串行；
- shrink 对 veth 删除做 pacing，避免 RTNL contention。

这是 host network resource pool，不是完整 netns/PodSandbox pool。

### 8.7 rootfs reuse 和 lazy data path

- 相同 RootfsConfig 合并并发首次 materialization；
- 已存在 rootfs 共享对象并 refcount；
- additional S3/OCI mount 按 key/url refcount；
- 默认 Python runtime EROFS 已烘焙到 node image；
- Nydus 成功时 lazy read；
- distill-fs 支持 raw/Nydus 按需读取、cache、chunk dedup、peer；
- 普通 OCI fallback 仍需完整 fetch/extract/overlay mount。

这些机制缩短 image/rootfs 路径，但不等于 application process 或 memory snapshot 复用。

### 8.8 每个 Sandbox 仍 fresh runsc create

当前每次 Start 都创建：

- Sandbox ID/metadata；
- OCI bundle/config；
- writable memory overlay；
- runsc sandbox/gofer/control socket；
- actor runtime/process；
- raw network socket；
- optional DNAT。

runsc handler 每次执行 runsc create，之后通过 gVisor control socket和 uRPC配置 link/route并 StartRoot。

因此 AKernel 当前没有：

- 完整 Ready Sandbox Pool；
- golden execution snapshot；
- Actor→Worker remap；
- pause/resume；
- checkpoint/restore；
- live fork。

README 的 40 ms、fork-based launch 和 checkpoint/restore有星号，并明确未在 v0.1.0 提供；sandboxd template_id 也明确被开源 runtime path 忽略。

### 8.9 command fast path

Sandbox 创建后，commands.run/start 不再调用 Kubernetes或 sandboxd Exec。它直接 RPC 到远端 _SandboxInstance actor，由 actor 内 subprocess.run/Popen 创建进程。

这与常驻 Pod 内 envd/runner 类似：steady-state command 很短，但 CommandHandle 只是 actor instance + PID；Popen、输出和 reader thread 在 actor 内存 map 中，不是 durable AProc。

## 9. Ray、openYuanRong/AKernel 与 Agent Substrate 的 Actor 是同一个东西吗

### 9.1 直接答案：同一家族，不是同一抽象

三者都使用了 Actor 家族的几个核心思想：

- 一个有身份的长期逻辑对象；
- 对象内部有可变状态；
- 调用方不直接访问内部内存，而是通过 handle、RPC 或路由地址交互；
- 系统负责把逻辑对象放到远端进程、节点或 sandbox 上；
- 不同 Actor 可以并行运行。

但它们的**抽象边界不同**：

~~~text
Ray Actor
  = user-defined stateful class / process
  = remote methods + futures + per-Actor method chain

AKernel/openYuanRong instance
  = sandbox 内的 stateful Python RPC management object
  = _SandboxInstance + command/filesystem/tunnel methods
  = 当前 AKernel Sandbox 的控制入口和生命周期锚点

Agent Substrate Actor
  = 一整个 agent-like workload / sandbox 的逻辑身份
  = lifecycle record + snapshot + current Worker assignment + HTTP route
~~~

因此：

- 如果比较“远程有状态对象编程模型”，AKernel 的 openYuanRong instance 更接近 Ray Actor；
- 如果比较“可长期存在、idle 时释放物理资源、事件到达时重新激活的 Agent execution environment”，Agent Substrate Actor 比 Ray Actor 和当前 AKernel instance 都更高一层；
- Agent Substrate 使用 Actor 这个名字，不代表它实现了 Ray 的 remote method、future、mailbox 或单 Actor 串行方法语义。

### 9.2 Ray OSDI 2018 Actor 的严格基线

Ray 论文没有发明 Actor 这一通用概念；论文 Related Work 已把 Orleans、Akka、Erlang 等列为既有 Actor systems。Ray 的贡献是把 Actor 与 task-parallel abstraction 统一到同一个 dynamic task graph 和 execution engine，而不是首次提出 Actor。

本地 Ray PDF 共 18 个物理页，物理页 1 是 USENIX 封面；关键证据如下：

| 位置 | Ray 论文直接给出的语义 |
|---|---|
| PDF 5 / 论文 564，§3.1 | Actor 是 stateful computation；暴露 remote methods；同一 Actor 的方法 serially executed；调用立即返回 future；handle 可传给 task 或其他 Actor |
| PDF 6 / 论文 565，§3.2 | 同一 Actor 的连续方法由 stateful edges 串成 chain，显式表示调用顺序和共享 mutable state 的依赖 |
| PDF 7 / 论文 566，§4.1 | Actor 是显式实例化的 stateful process；与 stateless worker 不同，方法依赖前一次方法执行后的状态 |
| PDF 9 / 论文 568，§5.1 | Actor 一旦 placed，正常执行时不能像 task 一样把 computation 移到大对象所在节点；Actor 是 coarse-grained placement |
| PDF 11 / 论文 570，§5.1 | Actor 故障恢复使用 stateful-edge lineage、user-defined checkpoint 和 method replay；实验中 2,000 Actors 里受两节点故障影响的 400 Actors 在剩余节点重建 |

Ray Actor 最核心的接口是：

~~~python
actor = Class.remote(args)              # 返回 Actor handle
future = actor.method.remote(args)       # 非阻塞，返回 future
value = ray.get(future)                  # 等待结果
~~~

这里的 Actor state 是用户 class/process 内部状态。例如 Figure 3 的 Simulator Actor 保存 `self.env`，同一 Simulator 的多次 `rollout` 共享它；多个 Simulator Actors 则可以并行执行。

Ray 的 stateful edge 有两个作用：

1. 定义同一 Actor 方法的先后状态依赖；
2. 把 Actor method 纳入 lineage，使故障后可以从 checkpoint 恢复并 replay 后续 methods。

这与完整进程/VM snapshot 有本质区别：Ray 2018 的 checkpoint 是 user-defined state checkpoint，恢复是 checkpoint + method replay，不是保存 Linux 进程树、anonymous memory、文件系统、设备和网络连接的透明镜像。

Ray 论文也没有定义：

- idle Actor 自动 suspend 并释放一个 warm slot；
- 外部 HTTP/DNS 可路由的永久 Actor identity；
- gVisor/microVM isolation；
- per-Actor cgroup hard isolation；
- 任意外部数据库或网络副作用的端到端 exactly-once；
- 正常运行中的 live migration。

所以 Ray 是“应用级 stateful computation + method lineage”基线，不是 Agent Substrate 式完整 sandbox lifecycle substrate。

### 9.3 openYuanRong 与 Ray：可以证明兼容映射，不能证明历史来源

AKernel 使用的是 openYuanRong 原生接口：

~~~python
@yr.instance
class _SandboxInstance:
    ...

handle = _SandboxInstance.options(options).invoke(cwd=cwd)
yr.get(handle.ping.invoke())
~~~

它没有 `import ray`，也没有 `import ray_adapter`。因此 AKernel 对 Ray 没有直接代码依赖。

不过，openYuanRong 官方 `ray-adapter` 的固定源码给出了非常明确的**语义映射**：

- Ray `ActorClass` 通过 openYuanRong `InstanceCreator.create_from_user_class` 构造原生 stateful instance；
- Ray `ActorHandle` 包装 openYuanRong `InstanceProxy`；
- Ray `ActorMethod.remote()` 转成原生 `MethodProxy.invoke()`；
- Ray `ActorClass.remote()` 转成 `InstanceCreator.invoke()`；
- Ray `num_cpus`、`max_concurrency`、named actor 和 detached lifetime 被转换成 openYuanRong `InvokeOptions`。

这证明 openYuanRong stateful instance 足以作为 Ray Actor API 的实现底座，也说明二者在 class→remote instance→handle→method future 这一编程模型上高度同构。

但“高度同构”不等于“历史上来自 Ray”：

- Actor model 本身早于 Ray；
- openYuanRong 原生文档称其为 stateful function/instance，而不是把 Ray Actor 作为规范；
- `ray-adapter` 是兼容层，也可能是把 Ray API 映射到一个独立演化的 native instance system；
- AKernel 仓库没有设计历史或代码提交证据证明 `yr.instance` 是从 Ray fork 而来。

因此本文采用的准确表述是：

> AKernel 使用的是 Ray-like openYuanRong stateful instance；openYuanRong 存在源码可见的 Ray compatibility adapter，但仅凭 AKernel 与 adapter 源码不能断言其 Actor 概念的唯一历史来源就是 Ray。

还要保留版本边界：AKernel node/runtime image 下载 openYuanRong 0.9.1 binary/wheel，而公开 AKernel SDK 声明 `openyuanrong-sdk==0.7.51`；真正的 instance registry、method dispatcher、scheduler 和 retry implementation 不在 AKernel 仓库中。上游 `ray-adapter` 可以证明接口映射，不能替代对 AKernel 所打包精确版本的运行时证明。

### 9.4 AKernel 中 Actor 的真实角色

#### 9.4.1 它是 sandbox management facade，不等于 AI Agent 对象

AKernel 仓库中唯一的 `@yr.instance` 用户类是 `_SandboxInstance`。它保存：

- `_cwd`：默认工作目录；
- `_procs`：PID → `Popen`、命令、stdout/stderr chunk、reader thread 的内存 map。

它暴露的方法不是 Agent 的业务 `think/act/observe`，而是：

- filesystem read/write/list/remove/rename；
- command run/start/wait/list/kill/stdin；
- reverse tunnel server；
- ping/get_info。

用户的 AI Agent、shell、compiler 或 server 实际上是 `_SandboxInstance` 通过 `subprocess.run/Popen` 启动的**子进程**。所以 AKernel Actor 更像 sandbox 内的 control daemon/object；它给用户进程提供控制面，而不是把用户 Agent class 本身直接变成 Actor。

可见调用关系是：

~~~text
AKernel Sandbox Python wrapper
  -> openYuanRong InstanceProxy handle
  -> _SandboxInstance remote methods
  -> subprocess / filesystem inside gVisor sandbox

另一路：
  Sandbox physical ID
  -> frontend WebSocket / gateway
  -> PTY、大文件 copy、port forwarding
~~~

这也意味着 AKernel 的所有数据面操作并不都服从 Actor method 语义：PTY、大文件 copy 和 port forwarding 使用 physical instance ID 走 frontend/gateway，不经过 `_SandboxInstance` 的 method queue。

#### 9.4.2 创建完成语义与身份

`Sandbox()` 的源码路径是：

1. 初始化 openYuanRong client；
2. 构造 CPU/memory/limits、idle timeout、schedule timeout、rootfs、mount、port、name、detached 和 node affinity；
3. `_SandboxInstance.options(options).invoke(cwd=cwd)`；
4. `yr.get(handle.ping.invoke())`；
5. 将 handle 保存给 Filesystem/Commands，并查询 real/physical instance ID 给 PTY 和 gateway。

因此构造返回时，至少有一个能响应 Actor RPC 的远端 execution context；这与 Substrate `CreateActor` 只写入 SUSPENDED record 完全不同。

AKernel 当前可见四层 identity：

| identity | 用途 | 边界 |
|---|---|---|
| `handle.instance_id` | openYuanRong handle 内部 identity | 生成、稳定性和 generation 在预编译 runtime 内 |
| `get_real_instance_id(...)` | 得到 real/physical instance ID | public `Sandbox.id` 暴露这个值 |
| `name` + namespace `akernel` | named/detached instance lookup | public API 只有按 name 删除，没有完整 attach/rebuild Sandbox wrapper |
| PID | `CommandHandle` 定位 child process | 仅与当前 instance handle组合，没有 generation/stale-handle 防护 |

public `Sandbox.id` 的 docstring 明确称其为 physical sandbox ID，而不是类似 Substrate `Atespace/name/UID` 的长期逻辑 Actor identity。AKernel unit test也刻意区分 mocked `logical-id` handle 和 `physical-id` public ID。

这里不能过度断言：openYuanRong node agent到 sandboxd 的 Unix socket wiring、sandboxd fresh `runsc create` 都可见，但把一次 logical instance invoke 转换成哪个 sandboxd request的代码位于预编译 openYuanRong 中。因此“当前 public model把一个 Sandbox wrapper绑定到一个 instance handle和physical routing ID”可由源码证明；“所有情况下严格 1:1、故障后永不换 physical ID”不能由当前仓库证明。

#### 9.4.3 AKernel 不能直接继承 Ray 的严格串行语义

Ray OSDI 2018 明确规定同一 Actor methods serially executed。openYuanRong 最新文档也把普通 stateful function描述为默认单实例单线程，并允许显式配置 concurrency。

但 AKernel 的固定源码主动设置：

~~~python
options.need_order = False
~~~

代码注释说明 AKernel 假设“一个 sequential SDK client”，关闭 ordered RPC 是为了避免缺失 sequence number 阻塞后续请求。这带来三个结论：

1. AKernel 主要依赖 client 同步调用习惯，而不是主动要求 ordered RPC；
2. SDK 没有在 `Commands`、`Filesystem` 之上增加一把 per-Sandbox 总锁；
3. openYuanRong dispatcher 精确版本的 concurrency、FIFO、retry 和 delivery semantics 不在 AKernel 仓库，不能从 Ray 论文倒推。

大部分 foreground SDK 操作是 `invoke()` 后立即 `yr.get()`，因此一个普通单线程 client 表现为顺序调用。但 background command只等 `Popen`成功就返回；子进程和两个 output reader threads 可与后续 RPC 并行。多个 PTY session也走独立 WebSocket/thread。因此即便 Actor method dispatcher单线程，整个 AKernel sandbox内部仍然是多进程、多线程、可并发的 execution environment。

最安全的源码结论是：

> AKernel 将 `_SandboxInstance` 当作通常由一个同步 client 驱动的 stateful RPC object，但它显式关闭 ordered RPC；当前仓库不能证明跨多个 client/thread 的严格 FIFO、linearizability、at-most-once 或 Ray 2018 式 stateful-edge order。

#### 9.4.4 Actor state、资源和故障恢复

当前 `_SandboxInstance` 的可见 state 主要是 `_cwd` 与 `_procs` 内存 map。背景进程的 `Popen` 对象、输出 chunk 和 reader thread 都只存在于 Python heap；Actor/runtime 一旦重启，这些 handle state 无法从 AKernel 源码恢复。

文件 state 位于同一 gVisor sandbox 的 writable memory overlay，而不是 Actor 字段。sandbox 删除后，当前开源 runtime没有 execution snapshot把它恢复到另一 sandbox。

`get_info()` 也不是控制面 truth query：远端 Actor固定返回 `state=running`；CPU/memory 是 client 构造时缓存的 request。它不能替代 Substrate Actor record中的 versioned lifecycle status。

资源方面：

- openYuanRong `InvokeOptions`携带 per-instance CPU/memory request、limit 和 node affinity；
- sandboxd 可见路径把 CPU 转成 cgroup shares、memory 转成 hard limit；
- Actor启动的所有 child processes共享同一 gVisor sandbox和同一sandbox cgroup；
- 没有 per-method或 per-child resource domain。

故障方面：

- AKernel没有给 `_SandboxInstance` 配置 user checkpoint、method lineage或dedup log；
- `recover_retry_times`没有在当前 adapter中显式设置；
- sandboxd public lifecycle只有 Start/Delete/Wait/List/Stats，没有 pause/resume/checkpoint/restore；
- openYuanRong 进程和 sandboxd 的 systemd `Restart=always`只证明 daemon重启，不等于 Actor heap、Popen和filesystem execution state恢复；
- idle timeout的精确判定和 reclaim动作也在 openYuanRong黑盒内，不能把它写成 Substrate 式 snapshot-on-idle。

因此 AKernel Actor 当前是有状态 remote object，但不是 durable、可迁移、可 scale-to-zero 后无损恢复的 logical Actor。

### 9.5 Agent Substrate Actor 的真实角色

#### 9.5.1 它是 workload identity，不是 remote class

Substrate architecture 对术语有明确说明：Actor 是“an instance of an agent-like workload”。Actor 可以是 AI Agent，也可以是其他 bursty、stateful、需要 sandbox 的 application instance。

Control gRPC API提供：

- Get/Create/Update/Delete Actor；
- Suspend/Pause/Resume Actor；
- ActorSnapshot 与 tag；
- Actor/Worker list。

它没有 Ray 式：

- `Class.remote()`；
- `actor.method.remote()`；
- arbitrary user method descriptor；
- method future/object ref；
- per-method lineage graph。

业务请求通过另一路数据面进入：router从 Host解析 Actor identity，触发 `ResumeActor`，取得当前 Worker IP，然后把原始 HTTP traffic 送到 Worker内 atunnel，再转到 Actor workload。Actor 的业务接口由其容器/application自己定义，不由 Substrate Actor API定义。

#### 9.5.2 Actor record 与 Worker assignment 解耦

Substrate Actor record包含：

- `Atespace/name/uid/version/timestamps`；
- ActorTemplate identity；
- RESUMING/RUNNING/SUSPENDING/SUSPENDED/PAUSING/PAUSED/CRASHED/DELETING；
- 当前 ateom Pod namespace/name/IP/UID 与 WorkerPool；
- latest/in-progress/local snapshot；
- placement selector；
- volumes。

CreateActor只在 Redis/ValKey 中创建初始 `STATUS_SUSPENDED` record；它没有 Worker，也没有 application process。

Resume时：

1. 从 store读取 Actor和snapshot；
2. 从内存 Worker cache挑选 free Worker；
3. CAS写入 Worker assignment；
4. 把 Actor改成 RESUMING并记录当前 Worker；
5. atelet/ateom恢复或启动sandbox；
6. Actor变成 RUNNING。

Suspend时：

1. 按ActorTemplate的snapshot scope执行checkpoint；FULL保存可恢复的runtime state，DATA只保留durable data并在恢复时cold boot；
2. 创建独立 ActorSnapshot record；
3. 清空 Worker assignment；
4. 清空 Actor的Pod/IP/WorkerPool字段；
5. Actor仍以同一逻辑 identity保持 SUSPENDED。

这就是它与 AKernel current Actor最大的差别：Substrate 把 `Actor identity/state` 与 `current process/sandbox/Worker location`明确分开，允许 `A logical Actors >> W active Workers`。

#### 9.5.3 生命周期串行不等于 application request串行

Substrate 为同一 Actor 的 Resume/Suspend/Pause/Delete获取 Redis distributed lock；router还用 singleflight合并同一进程内的并发 Resume。它保证或试图保证的是**lifecycle workflow互斥和幂等恢复**。

这不是 Ray Actor mailbox：

- lock覆盖 lifecycle workflow，不覆盖已经转发到 workload的所有 HTTP request；
- router对每个请求独立执行route/resume逻辑；
- RUNNING后原始 HTTP traffic由Envoy/atunnel送入 application；
- application是否串行、并发、使用线程池或event loop由用户镜像决定；
- Substrate没有为业务请求建立stateful-edge method chain。

因此不能因为它叫 Actor，就宣称同一 Substrate Actor 的业务请求严格串行。源码只能证明同一 Actor lifecycle operation有分布式锁，以及同一 router进程内的并发 Resume被singleflight合并。

#### 9.5.4 Substrate snapshot 也不是 Ray lineage

Substrate的FULL模式保存gVisor或Cloud Hypervisor sandbox snapshot，目标是恢复一整个execution environment；DATA模式只保存durable data并cold boot。具体能覆盖哪些rootfs、memory、external volume和network state取决于sandbox runtime与snapshot scope，不能把所有配置都写成full-memory restore。

Ray 2018保存的是 user-defined application checkpoint + method lineage；失败时 replay methods。两者分别优化：

- Ray：重建应用级 stateful computation和其输出依赖；
- Substrate：idle/事件驱动 workload 的物理资源释放与完整 sandbox reactivation。

两者都叫“Actor恢复”，但恢复协议、状态覆盖面、副作用风险和成本模型完全不同。

当前 Substrate 也不能被描述为 Ray 式自动 Actor reconstruction：如果一个仍绑定 RUNNING/RESUMING Actor 的 Worker消失，syncer会把该 Actor标为 `CRASHED`并清除Worker位置；Resume的合法前置状态只有SUSPENDED或PAUSED，不包含CRASHED。已经成功发布的durable snapshot可供后续显式流程使用，但源码没有在Worker failure后自动从最新snapshot拉起Actor。

### 9.6 三者相同点与差异矩阵

| 维度 | Ray OSDI 2018 Actor | AKernel/openYuanRong Actor | Agent Substrate Actor |
|---|---|---|---|
| 抽象层 | application stateful class/process | sandbox内 management RPC object | logical agent-like workload/sandbox |
| 用户定义对象 | `@ray.remote class` | AKernel固定 `_SandboxInstance` | ActorTemplate定义的OCI workload |
| 创建API | `Class.remote()` | `@yr.instance` + `.options().invoke()` | `CreateActor`写SUSPENDED record |
| 业务调用 | `actor.method.remote()` | handle method `.invoke()`；AKernel多用`yr.get`同步等待 | HTTP/网络流量经router/atunnel进入应用 |
| 返回值 | future/ObjectRef | openYuanRong object ref + `yr.get` | 普通application HTTP response；Control API返回Actor record |
| 单Actor顺序 | 论文明确methods串行/stateful edge | AKernel显式`need_order=False`；精确dispatcher语义不可审计 | 只串行化lifecycle workflow，不保证业务请求串行 |
| 私有state | user class/process memory | `_cwd`、`_procs`及同sandbox filesystem/processes | workload runtime/data；FULL可保存runtime state，DATA只保存durable data |
| identity | 可传递Actor handle；论文未定义永久外部identity | instance handle + optional name；public暴露physical ID | Atespace/name/UID/version，独立于当前Worker |
| physical binding | Actor process；正常运行时coarse placement | public wrapper绑定instance handle/physical routing ID；严格映射在YR黑盒 | RUNNING时绑定Worker；SUSPENDED时无Worker |
| idle scale-to-zero | 无 | idle_timeout可配置，但无可见snapshot resume | 核心能力：checkpoint后释放Worker |
| 恢复方式 | user checkpoint + method replay | 当前AKernel源码无Actor execution C/R | gVisor/CH sandbox snapshot restore |
| 正常重映射 | 没有idle Worker remap语义 | 没有公开logical Actor→new sandbox remap | Resume可选择新的eligible Worker |
| 故障模型 | GCS lineage重建；外部副作用边界未解决 | 预编译YR语义不可审计；localActor state不durable | version/lock/workflow + snapshot；仍处于early阶段 |
| placement资源 | task/actor resource requirements | per-instance CPU/memory/affinity options | ActorTemplate/sandboxClass/selector匹配WorkerPool shape |
| isolation | 论文不提供gVisor/microVM isolation | gVisor sandbox + sandbox cgroup | gVisor或microVM sandbox in Worker Pod |
| 数据面寻址 | computation内Actor handle | handle RPC + physical ID gateway/PTY | Actor DNS/Host identity，由router动态解析当前Worker |
| 复用粒度 | task/actor execution engine，不是idle slot复用 | cgroup/veth/rootfs复用，instance仍需execution context | 完整Worker slot在不同Actor之间复用 |
| AI Agent关系 | Actor可实现simulator/trainer/server，但不是专门Agent sandbox | AI Agent通常是Actor启动的child process | Actor就是一份agent-like workload instance |

最容易混淆的是 Worker 一词：

- Ray worker 是执行 stateless remote task 的进程；
- Substrate Worker 是预启动、一次只承载一个 active logical Actor 的完整物理 slot；
- AKernel当前没有与Substrate完全等价的完整 warm Worker abstraction，主要池化的是cgroup/veth/rootfs等低层resource。

### 9.7 为什么这个区别对 AKernel 设计重要

如果把三者都简单写成“Actor”，会产生四个错误推论：

1. **错误推论：有 Actor handle 就有 durable Agent identity。**  
   AKernel public ID当前是physical ID，CommandHandle又只是instance handle + PID；这不能替代Substrate Actor的logical UID/version和snapshot lineage。

2. **错误推论：Actor天然保证所有Agent请求串行。**  
   Ray 2018保证单Actor method chain串行；AKernel关闭ordered RPC；Substrate仅对lifecycle加锁。三者不能互相套用。

3. **错误推论：Actor故障恢复等于sandbox恢复。**  
   Ray checkpoint/replay、openYuanRong instance restart和gVisor/microVM snapshot是三套不同机制。尤其不能用daemon `Restart=always`替代execution state C/R。

4. **错误推论：YuanRong Actor已经提供Substrate式idle multiplex。**  
   AKernel当前代码能创建、调用和terminate instance，也有idle timeout参数，但没有可审计的snapshot-on-idle、Worker release、logical-to-physical remap和request-triggered restore链。

对 AKernel 更清晰的未来分层是：

~~~text
Logical Agent/AProc identity
  - durable name/UID/generation
  - policy, provenance, lifecycle, checkpoint lineage
  - independent of node and sandbox incarnation

Physical Worker/Sandbox incarnation
  - gVisor/Kata/microVM execution slot
  - CPU/memory/cgroup/network/rootfs
  - may be pooled, replaced or remapped

In-sandbox control actor/process
  - command/files/PTY/tunnel RPC
  - launches and supervises child AProcs
  - incarnation-local state
~~~

当前 `_SandboxInstance` 很适合保留在第三层；需要补的是前两层之间的稳定 logical handle、generation检查、snapshot/restore和Actor→Worker映射，而不是把现有 YuanRong instance handle重新命名就视为能力已经存在。

### 9.8 最终判断

对“AKernel Actor 与 Agent Substrate Actor 是不是一个东西”的准确回答是：

> 不是。AKernel当前使用的openYuanRong Actor/instance更接近Ray的应用级远程有状态对象，但在AKernel中它被固定成一个sandbox管理facade；Agent Substrate Actor则是一整个可暂停、可快照、可脱离Worker存在并由HTTP事件重新激活的逻辑workload。二者共享Actor的identity/state/remote-interaction思想，却在调用接口、状态边界、顺序语义、持久性和物理复用粒度上不同。openYuanRong的ray-adapter证明它可以承载Ray API，但不能单凭这一点断言AKernel Actor的历史来源只来自Ray，也不能把Ray或Substrate的能力自动归给当前AKernel实现。

## 10. AKernel 与 Substrate 的相同点

### 10.1 都采用两层时间尺度

Kubernetes：

- node/control component provisioning；
-长期 Pod lifecycle；
-集群基础设施。

专用控制面：

- per-Actor/per-Sandbox create；
- placement；
- node runtime operation；
- routing/exec。

### 10.2 都避免 per-execution Kubernetes object

高频执行单元不需要：

- Kubernetes Pod；
- kube-scheduler；
- kubelet Pod status；
- per-instance Service/Endpoint；
-通用 CNI/CSI controller chain。

### 10.3 都在 node 内提供专用 actuator

- Substrate：atelet + ateom；
- AKernel：openYuanRong node runtime + sandboxd。

二者都直接操作 gVisor/runtime、filesystem 和 network，而不是要求 kubelet为每个 Agent 创建 Pod。

### 10.4 都把慢操作预先准备或共享

- Substrate：Worker Pod、golden snapshot、runtime assets；
- AKernel：cgroup、veth/IP、local EROFS、rootfs mount/cache/chunk。

### 10.5 都重新承担控制面正确性

绕开 Kubernetes per-object reconciliation 后，需要自己处理：

- ID 和 generation；
- admission/capacity；
-并发 claim；
- rollback/cleanup；
- node failure；
- metadata persistence；
- routing；
- audit和 policy；
- stale handle；
-资源泄漏。

## 11. AKernel 与 Substrate 的差异

### 11.1 详细矩阵

| 维度 | Agent Substrate | AKernel v0.1 当前源码 |
|---|---|---|
| logical unit | Actor | openYuanRong instance / Sandbox |
| physical slot | Worker Pod | node 内新建 gVisor sandbox |
| Kubernetes unit | WorkerPool Deployment/Pod | master/frontend/node长期 Pod |
| create return | SUSPENDED logical record | actual actor ping完成 |
| active placement | 从预热 Worker cache选 slot | openYuanRong global scheduler选 node；算法不可见 |
| worker selection | sandboxClass/labels/local node + random | CPU/memory/affinity options可见；内部策略不可审计 |
| claim | Redis Worker CAS + Actor version | openYuanRong内部不可见；sandboxd reserve ID/local resources |
| node call | ateapi→atelet→ateom | openYuanRong node→sandboxd Unix socket |
| runtime action | gVisor/CH restore或boot | fresh runsc create |
| pool granularity | entire warm Worker slot | cgroup、veth/IP、rootfs mount/data |
| full state snapshot | gVisor/CH support | 不支持 |
| idle multiplex | Actor snapshot后 W slots服务 A≫W | idle Sandbox要么继续占资源，要么删除丢运行态 |
| routing | Actor DNS + Envoy + atunnel | Sandbox ID/port gateway与reverse tunnel |
| saturation | request parking + singleflight | schedule timeout/资源 max；无等价 parking |
| per-instance resource | WorkerPool shape为主 | SDK CPU/memory request/limit |
| dynamic state store | Redis/ValKey Actor/Worker | openYuanRong/etcd黑盒 + sandboxd node-local |
| user process handle | Actor service endpoint | Sandbox + process-like CommandHandle |
| durability | Actor/snapshot metadata较明确，仍 early | detached/name有限；CommandHandle/PID内存态 |
| K8s hot-path exception | informer、Secret GET、external volume | K8s node/pod resource watch；per-Sandbox路径不创建Pod |

### 11.2 Substrate 的优势

- logical Actor 与 Worker完全分离；
-完整 suspend/resume；
-预热的是可承载 sandbox 的完整物理 slot；
- snapshot 允许 idle state scale-to-zero；
-统一 routing/wakeup；
- bounded parking 和 same-Actor singleflight；
- Actor/Worker workflow源码可审计。

### 11.3 AKernel 的优势

- per-Sandbox CPU/memory request/limit；
-专用 fresh gVisor creation path；
-节点内部 FS 与 resource preparation 显式并行；
- cgroup/veth 粒度池化；
- rootfs lazy/cache/dedup/P2P方向更深；
- command/files/PTY接口适合交互式 sandbox；
-不要求 ActorTemplate/WorkerPool 这种固定 shape 才创建 Sandbox。

### 11.4 两者各自的缺口

Substrate：

- early development；
- auto-suspend-on-idle 尚未通用实现；
- scheduler 线性扫描；
- Redis/Worker cache consistency和 HA；
-外部 volume、security、identity尚未成熟；
- Worker pool exhaustion仍依赖慢 Kubernetes扩容。

AKernel：

- openYuanRong scheduler不可审计；
-无完整 Warm Sandbox/Worker pool；
-无 process-memory snapshot、C/R、fork；
-没有 logical Actor→physical Worker重映射；
-无 bounded request parking；
- CommandHandle/PTY不 durable；
-内部 Sandbox resource 对 Kubernetes不可见，资源记账闭环需自行保证。

## 12. Declarative Tax 在三套路径中的映射

| Tax 来源 | 传统 K8s per Pod | Agent Substrate | AKernel |
|---|---|---|---|
| per-unit API object | Pod/Job/CR | Redis Actor record | openYuanRong instance + sandboxd metadata |
| desired-state controller chain | Deployment/ReplicaSet/自定义controller | 仅 WorkerPool/Template低频走K8s；Actor同步 workflow | 仅长期系统Pod走K8s；instance由专用控制面 |
| scheduler | kube-scheduler per Pod | Worker cache + random free slot | openYuanRong scheduler选 node |
| node preparation | kubelet/CRI/CNI/CSI | Worker已存在；ateom restore sandbox | sandboxd准备resource/rootfs并 fresh runsc |
| routing convergence | Service/EndpointSlice/Ingress | Actor DNS + router + atunnel | gateway按 Sandbox ID/port |
| state persistence | etcd API objects | Redis + snapshot store | openYuanRong/etcd black box + local store |
| failure reconcile | Kubernetes controller |专用 lock/version/workflow | openYuanRong不可审计；sandboxd局部恢复 |
| burst backpressure | API/scheduler/controller queue | bounded parking + capacity error | schedule timeout + resource exhausted |
| governance | RBAC/admission/audit/finalizer成熟 | CRD用于coarse governance；Actor plane自建 |外层K8s治理；内部Sandbox需专用治理 |

关键结论：

> Substrate 和 AKernel 都减少了 Kubernetes declarative tax，但没有减少所有 control-plane tax。它们将税从“通用对象收敛”转换为“专用 scheduler、数据库事务、node RPC、runtime 和 state data plane”。

## 13. 为什么不能只比较 Create RPC

三者 create 的完成语义不同：

| 系统 | create 返回时 |
|---|---|
| Kubernetes POST Pod | object accepted/persisted；通常尚未 Running/Ready |
| Substrate CreateActor | SUSPENDED Actor record存在；尚未 claim Worker |
| AKernel Sandbox constructor | openYuanRong actor已能 ping；用户服务可能尚未ready |

公平终点必须是 time-to-useful-action：

~~~text
T0：client提交逻辑环境创建
T1：placement/capacity完成
T2：sandbox process/VM ready
T3：application readyz
T4：第一条有意义的命令或请求完成
TTUA = T4 - T0
~~~

同时报告 T0→T1、T1→T2、T2→T3、T3→T4。

## 14. 建议的源码验证 Benchmark

### 14.1 对照系统

1. Kubernetes Deployment replicas scale-out；
2. direct Pod create；
3. Ready Pod + pods/exec/常驻 runner；
4. Agent Substrate gVisor；
5. AKernel gVisor。

同 runtime 组尽量固定相同 runsc release。Substrate gVisor checkpoint路径可能需要特定 allow-connected-on-save 支持；若 binary不同，必须单列。

### 14.2 公平配置

-同一裸金属 host和kernel；
-同一 OCI image digest；
-同一 CPU/memory shape；
-同一 application readyz；
-同一 registry；
-同一 network policy能力；
-同一 volume durability语义；
-相同 cache cold/warm定义；
-外部 load generator；
-每套系统顺序运行并彻底清理。

专用路径的 network/storage功能更窄时，不能只报更低 latency而忽略能力差异。

### 14.3 并发矩阵

- 1、8、32、128、512、2000 concurrent logical environments；
- small image与大依赖image；
- node image cold/warm；
- snapshot local/remote；
- WorkerPool有余量、刚好满、完全饱和；
- Redis/object-store latency注入；
-应用初始化 0、100 ms、1 s、10 s。

2000 用于与论文相邻调度实验的规模对齐，但本实验数据不能反写成 Liu Declarative Tax 原始结果。

### 14.4 需要记录的 stage

Kubernetes：

- client request；
- API accepted；
- Pod created；
- Scheduled；
- PodSandbox ready；
- Initialized；
- ContainersReady；
- application ready；
- first response。

Substrate：

- router receive；
- lock acquired；
- Actor load；
- Worker scan；
- Worker CAS；
- Actor RESUMING；
- atelet RPC；
- manifest；
- snapshot download；
- OCI/assets；
- ateom restore；
- readyz；
- Actor RUNNING；
- route/first response。

AKernel：

- SDK invoke；
- frontend/master scheduler；
- node dispatch；
- sandboxd Start；
- ID reserve；
- FS prepare；
- cgroup allocate；
- interface allocate；
- runsc create；
- network uRPC；
- actor ping；
- first command/response。

openYuanRong内部无源码时，应依赖 OTEL/log timestamp或增加公开 instrumentation，不能用猜测填 stage。

### 14.5 控制面指标

- API Server request count/method/resource；
- etcd writes/watch events；
- controller queue depth；
- scheduler pending/bind latency；
- Redis commands、CAS conflict、lock wait；
- Worker cache size/relist；
- control-plane CPU/RSS；
- snapshot/object-store bytes；
- node runtime CPU/I/O；
- network setup calls；
- success、timeout、capacity rejection；
- p50/p90/p95/p99/p99.9和置信区间。

### 14.6 关键 ablation

Substrate：

- CreateActor only vs create+first useful action；
- golden snapshot on/off；
- local pause vs remote suspend；
- request parking on/off；
- same-Actor singleflight on/off；
- snapshot/OCI prepare serial vs parallel；
- Secret cache hit/miss；
- Worker cache scan与indexed scheduler原型。

AKernel：

- cgroup pool on/off；
- interface pool on/off；
- FS/resource parallel vs serial；
- local EROFS、Nydus、ordinary OCI；
- rootfs cache cold/warm；
- no full pool vs future Ready Sandbox pool；
- fresh runsc vs future checkpoint restore。

Kubernetes：

- Deployment scale vs direct Pod；
- direct Pod vs prewarmed Pod runner；
- default scheduler vs prebinding；
- image cold/warm；
- CNI/volume最小与生产功能配置。

### 14.7 要验证的假设

这些是实验假设，不是本文结果：

1. Substrate 在 Worker有余量时会显著降低 create-to-ready tail，因为不创建/调度Pod。
2. WorkerPool饱和后，优势会受 parking budget和Kubernetes异步补容限制。
3. AKernel cgroup/veth pool和并行prepare会改善fresh gVisor create，但仍受runsc create影响。
4. AKernel在大镜像小working-set时可能凭lazy rootfs降低TTUA。
5. Substrate在idle logical Actor密度上会明显优于当前无C/R的AKernel。
6. direct Pod相对Deployment能减少controller层级，说明 Declarative Tax需要按具体对象路径测量。

## 15. 对 AKernel 设计的建议

### 15.1 保留 Kubernetes，但只保存 coarse durable state

适合放在 Kubernetes：

- cluster/node capacity；
- long-lived system component；
- Worker class/runtime class；
- coarse ARealm policy；
-基础 RBAC、NetworkPolicy、certificate；
-低频 desired capacity。

不适合每次都写 CR status：

- command/stdout chunk；
- wake/sleep event；
-每次 tool call；
-高频 Worker assignment；
-短命 child process；
-token/event stream。

### 15.2 引入类似 Substrate 的 logical AProc/Worker 分离

当前 Sandbox.id 是 physical ID。论文 AProc 应增加：

- logical AProc ID；
- generation/incarnation；
- current physical sandbox；
- runtime capability；
- state/checkpoint ref；
- lifecycle state；
- owner ARealm；
- resource contract；
- route/mailbox；
- parent-child/lineage。

physical sandbox可以变化，logical handle不变。

### 15.3 在低层 resource pool 之上增加完整 Worker/Sandbox pool

当前 cgroup/veth/rootfs pool 已有价值，但仍 fresh runsc。下一层可以：

1. 预建固定 shape Worker slot；
2. restore/checkpoint sandbox；
3. idle AProc释放slot；
4.支持 Actor→Worker remap；
5.按 runtime/CPU/memory class建多个池。

不能把每个 CPU/memory组合都预热，否则 pool fragmentation会抵消收益。

### 15.4 补 checkpoint-on-idle

没有 C/R 时只能：

- keep：保状态但占资源；
- delete：释放资源但丢 execution state。

实现顺序可先从 gVisor C/R开始，再把：

- process state；
- mutable rootfs；
- terminal/session；
- runtime version；
- policy handle；
- object/workspace version

绑定到 durable APlane manifest。

### 15.5 增加 request parking 和 singleflight

-同 AProc并发 wake只允许一个 restore；
-有全局/tenant/realm parking上限；
-deadline/priority-aware；
-capacity unavailable时有界等待；
-满时明确 shed；
-旧 generation 的 wake被 fencing拒绝。

### 15.6 不要复制 Kubernetes 的全部 Declarative Tax

专用 Agent PCB 应保存最小正确性状态，不要把每个高频事件变成 etcd object update。建议：

~~~text
durable control record：
  identity / generation / lifecycle / placement / checkpoint / resource / policy

high-rate data plane：
  logs / output / token / tool events / filesystem chunks / memory pages
~~~

### 15.7 同时保留 Kubernetes 的成熟能力

旁路 per-Pod path 不代表放弃：

- admission；
- quota；
- authentication/authorization；
- policy；
- audit；
- owner/finalizer；
- rollout；
- failure reconciliation。

AKernel必须在 ARealm/AProc 层补回等价的最小正确性机制，否则只是更快的远程进程，不是可靠 Agent OS。

## 16. 最终判断

### 16.1 对 Liu 等论文的回应

Substrate 是论文所建议的 fast path 类型：

- declarative K8s管理粗粒度容量和治理；
-专用 control plane 管理高频 instance state；
-状态同步链从多controller/API object收敛缩成 Redis transaction + direct RPC；
- runtime slot提前准备。

它不是取消声明式收益，而是把声明式机制移出不适合它的细粒度 wake path。

### 16.2 对 The New Stack 标题的准确解释

“Kubernetes control plane sits in the wrong place”不应理解为 Kubernetes 不能运行 Agent。更准确的是：

> Kubernetes适合供给机器、长期Pod和治理边界；若把每个高频、短活跃、长idle、需要反复sleep/wake的Agent incarnation都编码为Pod/CRD，并让通用controller收敛，则控制面位于错误的时间尺度上。

“beside Kubernetes”就是保留 Kubernetes infrastructure plane，在旁边增加 Agent execution plane。

### 16.3 对 AKernel 的定位

当前源码支持的准确描述：

> AKernel 是 Kubernetes-hosted、process-oriented sandbox substrate。它通过 openYuanRong 的专用 frontend/scheduler/node path 将 per-Sandbox 创建移出 Kubernetes Pod lifecycle，并在 sandboxd 中实现 FS/host resource并行、cgroup/veth池和rootfs复用；但每个 Sandbox仍 fresh gVisor create，openYuanRong调度不可审计，且尚无 Substrate 式 logical Actor/Worker multiplexing、request parking或checkpoint/restore。

因此 AKernel 与 Substrate 方向相同、层级不同：

- AKernel 当前优化 fresh sandbox node fast path；
- Substrate 优化 stateful Actor 的长期 multiplex/suspend/resume；
- AKernel论文的 AProc/ARealm/APlane 若补齐，可以把两者组合成更完整的 Agent-native cloud OS。

## 17. 关键源码索引

### 17.1 Ray Actor 与 openYuanRong 兼容关系

| 主题 | 证据 |
|---|---|
| Ray 本地论文 | [Ray: A Distributed Framework for Emerging AI Applications](</mnt/u/hukeyang/AKernel/AKernel_Docs/References/Moritz 等 - 2018 - Ray A Distributed Framework for Emerging AI Applications.pdf>) |
| Ray Actor定义、handle、remote method、串行 | PDF物理页5 / 论文564，§3.1、Table 1–2 |
| stateful edge、method chain和lineage | PDF物理页6 / 论文565，§3.2、Figure 4 |
| Actor是stateful process | PDF物理页7 / 论文566，§4.1 |
| normal placement/locality边界 | PDF物理页9 / 论文568，§5.1 |
| checkpoint + replay恢复实验 | PDF物理页11 / 论文570，§5.1、Figure 11(b) |
| openYuanRong Ray兼容层说明 | [README.en.md](https://github.com/openyuanrong/ray-adapter/blob/fb131246f2469b1a6ab1d03d8a3c9cf7d78354d1/README.en.md#L1-L30) |
| Ray ActorMethod→YR MethodProxy | [actor.py](https://github.com/openyuanrong/ray-adapter/blob/fb131246f2469b1a6ab1d03d8a3c9cf7d78354d1/ray_adapter/actor.py#L123-L165) |
| Ray ActorClass→YR InstanceCreator | [actor.py](https://github.com/openyuanrong/ray-adapter/blob/fb131246f2469b1a6ab1d03d8a3c9cf7d78354d1/ray_adapter/actor.py#L168-L250) |
| openYuanRong stateful function概念 | [Key Concepts](https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/key_concept.html#stateful-functions) |
| openYuanRong stateful instance与方法 | [Stateful Functions](https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/development_guide/stateful_function/index.html) |
| openYuanRong concurrency文档 | [Concurrency](https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/development_guide/stateful_function/parallel-execution.html) |
| openYuanRong fault tolerance文档 | [Fault Tolerance](https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/development_guide/stateful_function/fault-tolerance.html) |

### 17.2 Declarative Tax 论文

| 主题 | 证据 |
|---|---|
| 本地 PDF | [The Gap Between Serverless Research and Real-world.pdf](</mnt/u/hukeyang/AKernel/AKernel_Docs/References/Liu 等 - 2023 - The Gap Between Serverless Research and Real-world.pdf>) |
| Chapter II | PDF/论文页 3–4 |
| OpenFaaS Figure 4 | PDF/论文页 4 |
| DOI | https://doi.org/10.1145/3620678.3624785 |

### 17.3 Agent Substrate

| 主题 | 本地源码 |
|---|---|
| 架构成熟度警告 | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:1) |
| Actor/Worker解耦与预热Pod | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:34) |
| focused control plane | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:52) |
| 为什么需要专用控制面 | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:148) |
| CRD/Redis分层 | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:205) |
| Actor术语是agent-like workload instance | [architecture.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/docs/architecture.md:27) |
| ActorRef与稳定mesh DNS | [actorref.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/internal/resources/actorref.go:25) |
| Control API只有Actor lifecycle/snapshot | [ateapi.proto](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto:23) |
| FULL/DATA snapshot scope | [ateapi.proto](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto:91) |
| Actor identity/status/Worker/snapshot字段 | [ateapi.proto](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/pkg/proto/ateapipb/ateapi.proto:161) |
| CreateActor只写SUSPENDED记录 | [create_actor.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/create_actor.go:99) |
| Redis SETNX Actor | [ateredis.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/store/ateredis/ateredis.go:414) |
| Resume workflow | [workflow.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow.go:167) |
| Worker claim/前向恢复 | [workflow_resume.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go:200) |
| Worker scheduler | [scheduling.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go:89) |
| Worker Redis CAS | [ateredis.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/store/ateredis/ateredis.go:639) |
| Worker内存cache | [workercache.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/workercache/workercache.go:15) |
| WorkerPool→Deployment | [workerpool_apply.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/atecontroller/internal/controllers/workerpool_apply.go:141) |
| atelet restore并行 | [main.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/atelet/main.go:493) |
| routing/wakeup | [extproc.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/atenet/internal/router/extproc.go:136) |
| atunnel HTTP reverse proxy而非Actor mailbox | [server.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/internal/atunnel/server.go:230) |
| singleflight/parking | [resumer.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/atenet/internal/router/resumer.go:89) |
| lifecycle Actor distributed lock | [workflow.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow.go:167) |
| suspend后释放Worker并清空location | [workflow_suspend.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go:216) |
| parking默认值 | [parking.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/atenet/internal/router/parking.go:26) |
| Secret 30秒cache和API miss | [workload_spec.go](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/cmd/ateapi/internal/controlapi/workload_spec.go:34) |
| 项目状态与demo数字 | [README.md](/mnt/u/hukeyang/AKernel/agent-substrate/substrate/README.md:19) |

### 17.4 AKernel

| 主题 | 本地源码 |
|---|---|
| SDK options与openYuanRong | [_openyuanrong.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:42) |
| 唯一`@yr.instance`类 | [_instance.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_instance.py:15) |
| Actor `_cwd`与`_procs`内存state | [_instance.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_instance.py:28) |
| foreground/background command实现 | [_instance.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_instance.py:190) |
| 关闭ordered RPC | [_openyuanrong.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:146) |
| invoke并等待ping | [_openyuanrong.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:203) |
| logical handle ID→real ID查询 | [_openyuanrong.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:210) |
| Sandbox physical ID | [sandbox.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/sandbox.py:218) |
| public ID明确是physical sandbox ID | [sandbox.py](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/sandbox.py:256) |
| SDK client依赖openYuanRong 0.7.51 | [pyproject.toml](/mnt/u/hukeyang/AKernel/akernel/sdk/python/pyproject.toml:29) |
| openYuanRong二进制下载 | [node.Dockerfile](/mnt/u/hukeyang/AKernel/akernel/builder/node.Dockerfile:127) |
| node runtime→sandboxd Unix socket | [yuanrong.service](/mnt/u/hukeyang/AKernel/akernel/builder/systemd_services/yuanrong.service:4) |
| node DaemonSet | [daemonset.yaml](/mnt/u/hukeyang/AKernel/akernel/deploy/akernel/charts/core/templates/node/daemonset.yaml:1) |
| node RBAC只有Node/Pod watch | [rbac.yaml](/mnt/u/hukeyang/AKernel/akernel/deploy/akernel/charts/core/templates/rbac.yaml:1) |
| master global scheduler端口 | [akernel_master.yaml](/mnt/u/hukeyang/AKernel/akernel/deploy/akernel/charts/core/templates/master/akernel_master.yaml:121) |
| sandboxd public API | [sandbox-api.proto](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:21) |
| template_id忽略 | [sandbox-api.proto](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:87) |
| Start FS/资源并行 | [server.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/server.go:816) |
| cgroup/interface并行 | [start_resources.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/start_resources.go:31) |
| cgroup pool | [cgroup.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/cgroupmanager/cgroup.go:140) |
| interface pool | [interface.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/networkmanager/interface.go:129) |
| rootfs和mount复用 | [fsmanager.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/fsmanager.go:106) |
| 每Sandbox fresh runsc | [client.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/runsc/client.go:75) |
| runtime handler无C/R | [handler.go](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/handler.go:32) |
| roadmap边界 | [README.md](/mnt/u/hukeyang/AKernel/akernel/README.md:33) |

### 17.5 既有 AKernel 调研

- [Kubernetes CRI/CNI/CSI启动管线与AKernel节点资源池](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/refs/surveys/20260801-170226-kubernetes-cri-cni-csi-pipeline-vs-akernel-resource-pooling.md)
- [从 pods/exec 到 Child AProc](/mnt/u/hukeyang/AKernel/AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/refs/surveys/20260724-172804-kubernetes-nested-agent-vs-akernel-process-handle-and-stateful-agent-lifecycle.md)

## 18. 参考资料与来源链接

1. Qingyuan Liu, Dong Du, Yubin Xia, Ping Zhang, Haibo Chen. The Gap Between Serverless Research and Real-world Systems. SoCC 2023. https://doi.org/10.1145/3620678.3624785
2. The New Stack. Kubernetes won the container decade. Google’s Agent Substrate wants the next one. https://thenewstack.io/kubernetes-ai-agent-runtime/
3. Agent Substrate fixed source. https://github.com/agent-substrate/substrate/tree/cbdeb7dbe003a55a16960a301bc595d9aa38b1ad
4. AKernel fixed source. https://github.com/akernel-dev/akernel/tree/5684c858cd835db053adef1265419af64e67bd04
5. Kubernetes Controllers. https://kubernetes.io/docs/concepts/architecture/controller/
6. Kubernetes API Concepts. https://kubernetes.io/docs/reference/using-api/api-concepts/
7. Kubernetes Custom Resources: declarative API boundaries. https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis
8. OpenStack Performance Docs. Results of measuring API performance of Kubernetes. https://docs.openstack.org/developer/performance-docs/test_results/container_cluster_systems/kubernetes/API_testing/index.html
9. Dong Du et al. Catalyzer: Sub-Millisecond Startup for Serverless Computing with Initialization-Less Booting. ASPLOS 2020. https://doi.org/10.1145/3373376.3378512
10. Simon Shillaker, Peter Pietzuch. Faasm: Lightweight Isolation for Efficient Stateful Serverless Computing. USENIX ATC 2020. https://www.usenix.org/conference/atc20/presentation/shillaker
11. Aleksandar Cvetković et al. Dirigent: Lightweight Serverless Orchestration. SOSP 2024. https://doi.org/10.1145/3694715.3695966
12. Philipp Moritz et al. Ray: A Distributed Framework for Emerging AI Applications. OSDI 2018. https://www.usenix.org/conference/osdi18/presentation/moritz
13. openYuanRong. Key Concepts: Stateful Functions. https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/key_concept.html#stateful-functions
14. openYuanRong. Stateful Functions and Concurrency. https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/development_guide/stateful_function/parallel-execution.html
15. openYuanRong. Stateful Function Fault Tolerance. https://docs.openyuanrong.org/en/latest/multi_language_function_programming_interface/development_guide/stateful_function/fault-tolerance.html
16. openYuanRong ray-adapter fixed source. https://github.com/openyuanrong/ray-adapter/tree/fb131246f2469b1a6ab1d03d8a3c9cf7d78354d1
