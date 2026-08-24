# Agent Substrate 源码深读与设计动机分析 —— 修正"范式错位"快速结论

> 生成:2026-08-24(本地) / 20260823T160043Z(UTC)
> 源码:`/home/keyang/AgentInfraWorkspace/google/substrate`(Go module `github.com/agent-substrate/substrate`,API group `ate.dev/v1alpha1`,Google 出品但标注 "not an officially supported Google product")
> 前序:[BareMetal 部署与 C/R 验证](20260820T133500Z-baremetal-agentenv-substrate-deployment-cr-verification.md)、[PR#14228 三系统对比](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md)
> 方法:全量源码阅读(cmd/pkg/internal/docs/demos/manifests),所有论断带 file:line 证据

## 0. 为什么要重读:此前结论被 README 打脸

前一轮快速调研我给出的结论是"**范式错位,不建议投入**",依据是四条实测/印象:HTTP request/response Actor 模型、atenet-router 18000 端口 Host 头路由、WorkerPool 固定 replicas、parking 5s 后 503 + timeoutSeconds 上限 30s。但 README 明确写着:

> **Claude Code & CodeX:** Support for high-density, stateful coding environments that preserve terminal and filesystem state across sessions.

且生态一节指向 [google/ax Agent Executor](https://github.com/google/ax)——"distributed agent runtime that demonstrates building a secure, hyper-scalable agent harness on Agent Substrate"。如果范式真错位,Google 不会专门给它写 harness。本次源码深读逐条核实,**四个论断三个不成立或需重要修正**,"范式错位"的定性必须撤回。

## 1. 四条论断核实结果

| 原论断 | 判定 | 修正(证据) |
|---|---|---|
| atenet-router 18000 端口 Host 头路由 | **半对** | Host 头路由正确;但 18000 是 router pod 内 **xDS server 端口**(`--port-xds=18000`,`manifests/ate-install/atenet-router.yaml:175`),供 Envoy sidecar 消费,不是客户端入口。真正入口是 `svc/atenet-router` 80→8080/443→8443(`atenet-router.yaml:337-352`),文档统一用 `kubectl port-forward svc/atenet-router 8000:80`。当时把 18000 当入口是误读 |
| WorkerPool 固定 replicas | **错** | `WorkerPool.spec.replicas` 带声明式 **scale subresource**(`pkg/api/v1alpha1/workerpool_types.go:115`),就是为 HPA 预留的;`demos/autoscaled-workerpool/` 附带完整 HPA + prometheus-adapter 栈,按外部指标 `ate_workerpool_workers{state=assigned}` 扩缩。正确表述:**Substrate 自身不自动扩缩,但插上了 K8s 自动扩缩的口**(`docs/architecture.md:94-96` 也承认"may be sufficient or might not") |
| actor 空闲 5s 被 park 后 503 | **错(张冠李戴)** | 代码里**不存在任何基于空闲的自动挂起**;suspend/pause 都是显式 RPC(`SuspendActor`/`PauseActor`,`pkg/proto/ateapipb/ateapi.proto:38-41`)。5s 是**入站请求 parking 预算**:请求到达时若 actor 抢不到空闲 worker,请求被驻留重试最多 `DefaultParkedRequestBudget = 5s`(`cmd/atenet/internal/router/ingress/parking.go:29`,重试 100ms 起 ×1.1 退避),超时才 503 "no free workers available"(`docs/request-parking.md:42-44`)。被 suspend 的 actor **永远不会因挂起而 503**——router 对每个请求自动触发 resume(见 §3) |
| timeoutSeconds 上限 30s 放不下长会话 | **错(读错字段)** | 唯一的 `timeoutSeconds` 是 `ContainerReadyz.TimeoutSeconds`——**就绪探针轮询超时**,默认 30s、min 1、**max 3600s**(`pkg/api/v1alpha1/actortemplate_types.go:177-180`)。真正的请求级期限是 Envoy 端到端 route timeout,默认 10s(`defaultRouteTimeout`,`cmd/atenet/internal/router/xds.go:109`),但可由 operator 经 `--route-timeout` 调整,manifest 注释**明确预期长 LLM 生成**并建议 `- "--route-timeout=5m"`(`atenet-router.yaml:197`);`--route-timeout` 帮助文本原话:"a harness relaying an LLM completion holds the request open for the whole generation"(`cmd/atenet/internal/router/cmd.go:76`)。Envoy 5min stream-idle 默认也被显式中和(`xds.go:111-122,236-253`) |

**教训**:四条里有三条是"把部署观察/字段名当成了设计语义"。部署实测看到的 503 是 pool 饱和 + parking 预算耗尽的表现,不是"空闲即死";30s 是探针默认值,不是会话上限。

## 2. 架构剖析

### 2.1 组件图(全部 `cmd/` 下,一 binary 一目录)

```
控制面(ate-system):
  ateapi        "大脑":gRPC Control API;状态存 ValKey/Redis;resume/suspend/pause 工作流引擎;
                调度器;actor 身份 JWT/证书签发
  atecontroller K8s controller-manager:把 WorkerPool CRD reconcile 成一个 worker Deployment(SSA,
                workerpool_apply.go:70-163);ActorTemplate(golden snapshot 编排)、NetworkPolicy
  atenet        路由/网关进程,一 binary 两个部署(atenet-router 入站 / atenet-egress 出站);
                Envoy ext_proc(gRPC)+ xDS server;备选数据面 agentgateway
  podcertcontroller 内部 CA:pod 身份/服务 DNS 证书(SPIFFE 风格,trust domain cluster.local)
  kubectl-ate   CLI/kubectl 插件:create/get/suspend/resume/pause/delete/logs/snapshots——
                无 exec、无文件上传下载、无 port-forward-to-actor 子命令
节点面:
  atelet        DaemonSet"牧羊人":拉 OCI 镜像建 bundle;snapshot 与对象存储之间的上/下行流
                (sparse-extent + zstd 流,magic "ATESPRSE",ategcs/sparsezstd.go:27-33)
  ateom-gvisor / ateom-microvm   worker 容器镜像,一个 worker pod 一个 ateom:
                RunWorkload/CheckpointWorkload/RestoreWorkload + atunnel 入/出站 server
```

### 2.2 最关键的设计决策:Actor 不是 K8s 资源

CRD 只有 4 个:`ActorTemplate`(不可变 actor 版本:digest-pinned 镜像、快照配置、SandboxClass=gvisor|microvm)、`WorkerPool`、`SandboxConfig`、`CSIDriverConfig`。**Actor、Worker、Atespace、ActorSnapshot 是 Redis/ValKey 里的数据库记录**(`ateapi.proto:168-232`)。`docs/architecture.md:225-254` 给了明确理由:actor 的创建/销毁/状态迁移频率太高、目标规模太大("Scale: Target: 1 billion"),塞进 kube-apiserver 会压垮 API server——这是"Dynamic Instance State (Database-based)"一节的原意。actor FQDN:`<actor>.<atespace>.actors.resources.substrate.ate.dev`。

### 2.3 Actor↔Worker:时间复用,一 worker 一 actor

- Worker = WorkerPool 生成的 K8s Pod,内含 ateom 容器;actor 的容器作为**沙箱子容器**跑在该 pod 里(gVisor sentry 或 Kata microVM guest),外加一个 pause 式根容器持有 sandbox namespace(`cmd/atelet/main.go:1349-1405`)。
- **排他占有**:atunnel 强制 "There can be only one active actor per worker"(`internal/atunnel/ingress.go:180-189`)。
- **调度**:ateapi 侧快路径**完全绕过 K8s scheduler**——过滤 STATE_ACTIVE、未分配、sandboxClass/label 匹配(+ paused 的 RequiredNodes 钉节点)后**均匀随机**选一个(`cmd/ateapi/internal/scheduling/scheduling.go:110-133`)。
- **"Teleport" = 整沙箱全状态 checkpoint/restore,不是热迁移**:
  - gVisor 路径:`runsc checkpoint/restore` 整个 sentry + `runsc fscheckpoint` 处理 durable 目录(`cmd/ateom-gvisor/runsc.go:166-191, 237-272`);**需要打过补丁的 runsc**(`-allow-connected-on-save`,`runsc.go:150`,`docs/architecture.md:321`)。
  - microVM 路径(Kata on **Cloud Hypervisor**):Full scope = pause → file:// 快照(config.json+state.json+sparse memory-ranges)→ 拆掉 VMM;根文件系统写在 guest RAM 的 tmpfs overlay 上,内存快照顺带捕获(`cmd/ateom-microvm/checkpoint.go:34-52`)。restore 用 **userfaultfd 按需调页**:"idle restored actor holds its working set rather than its whole snapshot — 16MiB against 158MiB"(`restore.go:34-38`)。
- **Pause vs Suspend**:pause 快照留在本节点(快恢复、RequiredNodes 钉住);suspend 上传对象存储、任意节点可恢复(`ateapi.proto:35-41`)。
- **挂起的 actor 成本 = 一条 Redis 记录 + 对象存储里的压缩快照字节。零 pod/CPU/RAM。**这就是 250 actor 压 8 pod 的全部秘密——超卖是**时间复用**而非同驻。

### 2.4 Snapshot 语义与 ActorTemplate

ActorTemplate 不可变且镜像必须 digest-pinned("changing the image invalidates snapshots",`actortemplate_types.go:109`)——因为 golden snapshot 只对同一镜像+配置有效。`SnapshotsConfig`:`onPause/onCommit` scope=Full(内存+fs delta)|Data(仅 durable 目录),`onResume`=ColdBoot|Golden(golden snapshot 即模板级预热快照,`docs/architecture.md:181-185`,实测 ~400ms 冷启动)。快照传输是 sparse-extent + zstd 流(对象存储 GCS/S3,kind 里用 rustfs)。

## 3. 网络:唯一的入站面是"Host 头 → 自动 resume → 反代"

完整路径:

1. 客户端解析 actor DNS(atenet DNS 把它指到 router),HTTP 请求带该 Host。
2. Envoy(atenet-router sidecar)对**每个请求**调 Go ext_proc,且**只看 RequestHeaders**——body 原样流过(`cmd/atenet/internal/router/extproc/extproc.go:100-115`)。
3. ingress 从 Host 解析 actor ref,**每个非 RUNNING actor 的请求都自动触发 `ResumeActor`**(`cmd/atenet/internal/router/ingress/ingress.go:99-104`);同 actor 并发请求 singleflight 去重(`ingress/resumer.go:171`)。
4. ateapi resume 工作流:原子领取空闲 worker → mTLS 拨号节点 atelet → 下载快照 → ateom `RestoreWorkload` → atunnel Activate。
5. router 拿到 worker pod IP,改写路由头让 Envoy ORIGINAL_DST cluster 拨 `workerIP:443`(`ingress.go:126-147`),原 Host 保存在 `X-Ate-Original-Host`。
6. worker 侧 **atunnel** = "activation-aware HTTPS 反向代理":mTLS(验 router 的 podidentity 客户端证书)→ 反代到 actor 私有 veth 的 port 80;不匹配当前活跃 actor 的请求一律 421(`internal/atunnel/ingress.go:281-284`);目前只支持 actor 的 80 端口(`TODO(bowei)`, `ingress.go:131`)。

**没有 substrate 级的 exec/PTY/文件 API**。gRPC Control 服务只有生命周期操作(`ateapi.proto:25-83`);kubectl-ate 无 exec/cp。**出站方向倒是有任意 TCP**:actor 发起的 TCP 被 iptables REDIRECT 到 atunnel egress listener,经 `SO_ORIGINAL_DST` 恢复原始目的地(`original_dst_linux.go:23-27`),与 atenet-egress 建 **HTTP CONNECT** 隧道,用 actor 身份证书认证(`cmd/atenet/internal/router/egress/egress.go:85-132`)——所以 actor 可以主动连任意 LLM API,但入站只有 HTTP-over-Host。无任何 websocket/upgrade 专用代码;流式响应靠"通用 L7 代理链不碰 body"成立,长请求受 `--route-timeout` 约束(可调,见 §1)。

### Parking(503 的真相)

请求到达而 pool 暂时无空闲 worker → 进 parking lot 驻留重试(预算 5s,上限 1024 个驻留请求);lot 满则立即 503 "router at capacity"(`ingress.go:94-97`)。`docs/request-parking.md:26-29` 说清了动机:"such saturation is usually momentary: another actor suspends within milliseconds and frees its worker. Failing fast turns a sub-second blip into a user-visible error." 驻留请求能活过 router SIGTERM(优雅升级不丢请求)。resume 失败的重试策略表在 `docs/request-parking.md:84-92`:FailedPrecondition/Unavailable 重试、Aborted 恒重试、其余 fail-fast。

## 4. Claude Code / CodeX 到底怎么被承载

仓库里有直接证据:**`demos/claude-code-multiplex/`**——"three Claude-Code-driven agents sharing two Agent Substrate pods"。看 workload(`workload/Dockerfile` + `run.sh`):

```bash
# node:20-slim + npm install -g @anthropic-ai/claude-code,entrypoint 是自驱循环:
claude --print "${TASK}" 2>&1 || echo "... claude exited non-zero"
sleep "${INTERVAL_SECONDS}"
```

三个要点:

1. **actor 不是被"驱动"的,是自驱的**。用 `claude --print` 非交互一次性模式,任务文本经模板 `TASK` env var 在部署期注入;没有交互终端、没有 PTY。UI 的 "Give a task" 按钮**只是记录分配并渲染状态徽章**(经 ateapi gRPC ListWorkers/ListActors,`ui/server.go:359-434`),任务列表硬编码(`server.go:84`);输出经 pod logs 观察。
2. **持久性来自"整进程树被 checkpoint"**。terminal+fs 状态跨 suspend/resume 存活,是全沙箱 C/R 的副产品——CLI agent 作为 actor 主进程活着,sleep 的空闲窗口让 suspend 变得安全且廉价。README 说 "Substrate notices the inactivity and suspends the agent"——注意这是 demo 营销话术,substrate 代码里 suspend 仍是显式 RPC(demo 里由 CLI/operator 发起)。
3. **真正"harness 驱动 actor"的两种姿势**:
   - **gRPC 控制面**(`internal/ateclient`):CreateActor → ResumeActor →(放流量)→ SuspendActor。demos/sandbox、multiplex UI、kubectl-ate 全走这条。
   - **对 actor DNS 发 plain HTTP**:actor 自己实现任意 API。`demos/sandbox/main.go` 是范本——actor 里跑一个 `/process` endpoint,内部 `exec.CommandContext` 执行命令返回 `{stdout, stderr, exitCode}` JSON;REPL 客户端 POST 命令实现 `sandbox> ls -la`,客户端输入 `exit` 触发客户端侧发起 SuspendActor。**exec 是应用层约定,不是 substrate 能力**;文件系统持久化让"感觉像常驻会话"。
   - google/ax(Agent Executor)是被点名的正式 harness,代码不在本仓库。

## 5. 揣摩设计动机:为什么这么设计

### 5.1 出发点:agent 的时间结构

`docs/architecture.md:18-23` 原话:agents "spend most of their time waiting for input or events… the time they spend waiting can be unbounded… they often run untrusted logic, they are usually single-tenant instances, and there are a great many of them."——四句话四个设计推论:

| agent 特性 | 推出的设计 |
|---|---|
| 大部分时间在等(LLM/工具/用户) | 空闲即 snapshot 收走 worker → 时间复用超卖 |
| 等待时长无界 | 不能靠缩容杀掉,必须保状态 → 全状态 C/R 而非日志重放 |
| 运行不可信逻辑 | gVisor/microVM 真隔离,一 actor 一沙箱 |
| 海量单租户实例 | actor 状态进 Redis 不进 kube-apiserver;北极星指标直接写 "Scale: 1 billion" |

对照传统 K8s 的痛(`architecture.md:154-167`):idle Pod 仍占资源;kube-apiserver 撑不住百万级高频对象;"Pod 起几秒很好——当它要跑几小时时;但对于只跑毫秒到低秒级的负载,这个延迟不可接受"。**Substrate 本质是把"Pod"这个生命周期粒度从小时级压到请求级,同时用 C/R 保住小时级的状态。**

### 5.2 为什么入站是 request/response + Host 头路由

`architecture.md:59-63`:"In order to achieve on-demand resuming of actors, we need to be able to trap and inspect network traffic… a lightweight, substrate-aware network proxy which can trigger the resumption of actors as needed."——**唤醒一个挂起 actor 的唯一通用信号就是它的入站流量**。要在不改造 actor 的情况下"流量即唤醒铃",必须有一个能看 L7 头的代理在入口做 trap-and-inspect,于是 Envoy ext_proc + Host 头(=actor 身份)成了最小充分设计。suspend 一侧不需要 actor 配合(内核级整沙箱 checkpoint),resume 一侧用网络陷阱替代了 agent 配合——**对称地把"协作式生命周期"压成了"零侵入"**。

出站 CONNECT 隧道同理:不是为了限制,是为了**给每条出站流量盖上 actor 身份章**(egress ext_proc "authenticates the actor behind each egress CONNECT"),让审计/网络策略/多租户隔离在 mesh 层面免费获得。

### 5.3 为什么没有 exec/PTY/文件 API

读完代码我的判断:这是**有意的低意见(low-opinion)设计**,不是没做完。README 原话:"Agent Substrate is intended to be a low-opinion system… It is not an SDK for building agents, but rather a system for running them at scale." substrate 只承诺两件事:**生命周期(秒级 teleport)+ 位置透明路由(actor DNS 永不变)**。执行语义(exec、文件、流)留给两层:actor 内的应用(demos/sandbox 的 /process),或上层 harness(google/ax)。好处是控制面可以做到极小、极快、与任意框架正交;代价是每个 harness 都要自己发明一遍 exec-over-HTTP 约定。

### 5.4 为什么绑定 Kubernetes 却又绕过它的调度器

双轨:基础设施面(worker pod 的供给、扩缩、节点管理)留在 K8s(README:"consistent infrastructure management across all workload types… holistic infrastructure optimizations for RL scenarios that span agentic, inference and training cycles"——RL 训练/推理/agent 三类负载同池调度是战略动机);实例面(actor 的 claim/assign)走 ateapi 内存+Redis 快路径,均匀随机而非装箱优化——因为在"1 billion actors"的目标下,调度器自身的可扩展性比调度质量重要。这是典型的 control-plane-of-control-plane 分层:K8s 管"热的",Substrate 管"冷的+快的"。

### 5.5 诚实的未成熟标记

`architecture.md:3`:"Much of this architecture is aspirational, and is not yet implemented!";README:"not ready for production use… APIs are almost guaranteed to change";snapshot GC 未实现(`architecture.md:459`);gVisor 路径需要打补丁的 runsc。HPA 的 batchy scale-up / 300s 稳定窗也只是 demo 级调法。**按 demo/实验系统对待,勿当产品读。**

## 6. 修正后的结论:与我们的 harness 如何对接

撤回"范式错位"。准确的说法是:

**Substrate 提供的原子操作是"带全状态快照的秒级沙箱启停 + 流量驱动的位置透明唤醒";我们 harness 需要的 exec/文件/反向隧道面,Substrate 故意不提供,但留了两个对接点。**

对接 harbor-akernel(`AKernelEnvironment`)的实际映射:

| harness 需求 | AKernel 现状 | Substrate 对接方式 |
|---|---|---|
| 创建/销毁环境 | `Sandbox()` 构造/析构 | `CreateActor`/`DeleteActor`(gRPC,`internal/ateclient`) |
| exec 命令 | `sb.commands.run()` | **actor 内跑一个 exec-over-HTTP 约定服务**(照抄 `demos/sandbox/main.go` 的 `/process`);长输出/流式靠 body 流过代理链 + `--route-timeout=5m+` |
| 文件上传/下载 | `sb.fs.*` | 同上,actor 内 HTTP endpoint(tar/base64);或利用"fs 状态在快照里"经对象存储旁路 |
| 反向隧道(模型流量) | `AKERNEL_REVERSE_TUNNEL_TARGET` + atunnel | **不需要**——actor 出站任意 TCP(CONNECT 隧道),模型端点直连即可;比我们的反向隧道更顺 |
| PTY 长会话 | `sb.pty` | 无对应物;要么 codex/claude 用 `--print`/`exec` 自驱模式(官方 demo 就这么干),要么应用层实现 PTY-over-HTTP |
| 每任务镜像 | nydus_v2 私有 registry | ActorTemplate 要求 digest-pinned;golden snapshot 机制下换镜像=快照全部失效,12 个任务镜像要重做 golden |

**工程量评估**:写 `harbor.environments.substrate:SubstrateEnvironment` 的真实难点不是 API 面(生命周期 gRPC 很薄),而是 exec/文件约定服务要进任务镜像、以及 `--route-timeout` 等集群级参数不在 harness 掌控范围(我们用的是别人部署的集群)。相比之下 AgentEnv 路线(API 直接有 exec)对接成本更低;Substrate 的独特价值在于**时间复用超卖**(挂起零成本)——但我们的 benchmark 负载是"任务排队、沙箱满负荷跑小时级",几乎没有可复用的空闲窗口,**超卖收益在 frontier-swe 场景下接近零**。这推翻了我上次"范式不容"的表述,但"此场景不值得投入"的结论碰巧仍成立——理由从"接不上"修正为"接得上,但它优化的维度(空闲复用)不是我们的瓶颈"。

若未来场景变成"海量常驻 agent、低频交互、状态必须长存活"(例如托管成千上万个待命 coding agent),Substrate 的模型就是为此而生的,那时再投入。

## 7. 与既有 Survey 结论的衔接

- [PR#14228 对比文](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md)的核心判断被本次源码阅读**强化**:Substrate 的 suspend/cold 数百 ms 里控制面(ateapi→atelet→ateom、对象存储读写、worker claim)占大头;其 snapshot zstd 全量序列化在 GB 级可写层上会撞 O(S) 墙——源码确认 gVisor 路径就是 runsc checkpoint、microVM 路径是 CH 内存快照序列化,没有任何差分/外部化机制。"PR 式数据面 + Substrate 式控制面是自然组合"依然成立,且 gVisor 路径现在明确依赖**打过补丁的 runsc**(`--allow-connected-on-save`),说明上游化需求真实存在。
- 需要修正的前文表述:该文 §5 的 "饱和即 503(WorkerPool 固定)"应读作"parking 预算(5s)耗尽即 503,WorkerPool 可经 HPA 扩";"部署文档里的 18000 端口"应读作 xDS 端口,入口是 svc 80/443。

## 8. 关键文件索引

| 主题 | 位置 |
|---|---|
| 入站自动 resume | `cmd/atenet/internal/router/ingress/ingress.go:99-104` |
| parking 常量 | `cmd/atenet/internal/router/ingress/parking.go:29-43` |
| route timeout 默认与调参 | `cmd/atenet/internal/router/xds.go:109`;`manifests/ate-install/atenet-router.yaml:197` |
| 一 worker 一 actor | `internal/atunnel/ingress.go:180-189` |
| 随机调度快路径 | `cmd/ateapi/internal/scheduling/scheduling.go:110-133` |
| gVisor checkpoint/restore | `cmd/ateom-gvisor/runsc.go:150,166-272` |
| microVM 快照 + userfaultfd restore | `cmd/ateom-microvm/checkpoint.go:34-52`;`restore.go:34-38` |
| sparse+zstd 快照流 | `cmd/atelet/internal/ategcs/sparsezstd.go:27-33` |
| 出站 CONNECT + 身份认证 | `cmd/atenet/internal/router/egress/egress.go:85-132`;`internal/atunnel/egress.go:51-53` |
| Claude Code demo(自驱循环) | `demos/claude-code-multiplex/workload/run.sh` |
| exec-over-HTTP 范本 | `demos/sandbox/main.go`(/process endpoint) |
| WorkerPool scale subresource | `pkg/api/v1alpha1/workerpool_types.go:115` |
| HPA demo | `demos/autoscaled-workerpool/` |
| 设计动机文档 | `docs/architecture.md`(§18-23 agent 特性、§59-63 网络陷阱、§154-167 K8s 痛点、§225-254 DB-state 理由);`docs/request-parking.md` |
