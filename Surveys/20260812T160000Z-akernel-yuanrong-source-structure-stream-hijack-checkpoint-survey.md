# AKernel + yuanrong 源代码结构调研：调度链路与「model API stream 劫持 → checkpoint → resume」机制验证

> 调研时间：2026-08-12（Asia/Shanghai）
> 文档时间戳：`20260812T160000Z`
> 调研对象：`akernel/`（开源 v0.1.0）、`yuanrong/`（openYuanrong）、`AKernel_Docs/`
> 证据边界：本文以本地源码为"当前实现"证据，以项目 README/docs/outline 为"作者声明或设计意图"证据。未在目标硬件上运行，所有功能判定基于源码静态审计，不涉及性能复现。

---

## 0. 一句话结论

用户转述的机制——「sandbox 内 claude code 调用 model API，该 stream 被 sandboxd/proxy 劫持，触发整个 sandbox checkpoint，等 model API 返回完整结果后再 resume」——**在当前开源源码（akernel v0.1.0 + openYuanrong）中完全不存在实现，也没有任何文档描述**。它应当被视为 AKernel 开发者的**设计意图 / 路线图**，而不是已落地能力。

但调研中确认了两条**真实存在且重要**的相邻事实，容易和上述机制混淆：

1. **yuanrong 确实主动调用 sandboxd 启动 AKernel**（用户这条说法准确）。链路是 `RuntimeManager → SandboxdExecutor → gRPC(SandboxService) → sandboxd → runsc(gVisor)`。
2. **进程级 checkpoint/restore 在 yuanrong 侧是真实原语**：yuanrong 的 `SandboxdCheckpointOrchestrator` 会调用 sandboxd 的 `Checkpoint` / `Restore` gRPC。但**开源 sandboxd v0.1.0 的 proto 把 `Checkpoint/Restore/Register` 裁掉了**（只剩 6 个 RPC），yuanrong 内部 proto 仍保留 11 个 RPC。这是「内部完整版 vs 开源裁剪版」的 proto 分叉，是本文最重要的发现之一。

至于「model API stream 劫持」这条核心机制：源码里 `anthropic / openai / claude / model api / llm proxy / stream 劫持` 全部**零命中**；网络层只有 iptables NAT，没有任何 L7/HTTPS 拦截；三者（freeze、proxy、流识别）之间没有任何联动代码。

---

## 1. 调研范围与证据等级

### 1.1 主要源码范围

| 主题 | 入口 |
|---|---|
| yuanrong 调度与 executor | `yuanrong/functionsystem/functionsystem/src/runtime_manager/`、`function_master/global_scheduler/` |
| yuanrong ↔ sandboxd gRPC client | `yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp`、`sandboxd_checkpoint_orchestrator.cpp` |
| yuanrong 侧 proto 副本 | `yuanrong/functionsystem/proto/posix/sandbox_api.proto` |
| sandboxd API 契约 | `akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto`、`sandbox-api_grpc.pb.go` |
| sandboxd server | `akernel/src/sandboxd/internal/server/server.go` |
| sandboxd runtime（gVisor） | `akernel/src/sandboxd/pkg/runtime/runsc_handler.go`、`pkg/runtime/runsc/` |
| sandboxd 网络 | `akernel/src/sandboxd/pkg/networkmanager/bridge/network.go` |
| sandboxd 状态与 cgroup | `akernel/src/sandboxd/pkg/sandbox/status.go`、`internal/cgroupops/`、`pkg/cgroupmanager/` |
| distill-fs | `akernel/src/distill-fs/`（Rust） |
| 论文与设计意图 | `AKernel_Docs/Paper_outline.md`、`AKernel_Paper_Structure.md`、`Introduction_outline.md`、`outline_draft.md`、`akernel/README.md`、`akernel/AGENTS.md` |

### 1.2 证据等级

| 等级 | 含义 | 本文用法 |
|---|---|---|
| A | 固定 commit 源码/测试直接支持 | "已实现""未实现（源码无命中）""proto 分叉" |
| B | 项目 docs/README/outline 声明 | "设计意图""路线图""论文规划" |
| C | 推演、需进一步验证 | "若要落地需新增 X" |

---

## 2. 源代码整体结构

### 2.1 顶层布局

```
AKernel/                          # 论文工作区根目录（非单一 git 仓库）
├── akernel/                      # 开源 AKernel（SDK + 部署 + 节点组件源码子模块）
│   ├── src/sandboxd/             # 节点 sandbox 生命周期管理器（Go）
│   ├── src/distill-fs/           # 只读 FUSE 镜像懒加载（Rust）
│   ├── sdk/python/               # 用户面向 akernel_sdk + ak CLI
│   ├── builder/ deploy/          # all-in-one 镜像构建、Helm/Terraform
│   ├── README.md / AGENTS.md     # 项目说明与 agent 指引
├── yuanrong/                     # openYuanrong（分布式调度 + 数据 + 函数系统）
│   ├── functionsystem/           # 函数/实例调度与 runtime_manager（C++）
│   ├── frontend/                 # HTTP/WS 网关（Go）
│   ├── src/                      # libruntime（function 进程内 SDK 运行时，C++）
│   ├── datasystem/               # 分布式数据/对象系统
│   └── api/ sandbox-sdk/ go/     # 多语言 SDK
├── agent-substrate/              # 对照项目（Google AX Agent Substrate）
├── kvcache-ai/                   # 含 AgentENV 等对照系统
└── AKernel_Docs/                 # 论文大纲、调研、部署文档
```

`akernel/AGENTS.md`（=`CLAUDE.md`）明确：开源 akernel 仓库含 SDK、部署、构建工具与示例；节点运行时组件 `sandboxd` 与 `distill-fs` 以 git 子模块形式维护，all-in-one 镜像在专用 builder stage 编译它们，并下载官方 gVisor `runsc`（按 SHA-512 校验）。

### 2.2 sandboxd 内部结构（Go）

```
akernel/src/sandboxd/
├── api/runtime/v1/               # gRPC 契约：sandbox-api.proto + 生成码
├── cmd/sandboxd/ cmd/sbox/       # daemon 与 CLI 入口
├── internal/
│   ├── server/                   # gRPC handler（server.go：Start/Delete/...）
│   ├── langrtmanager/            # 语言运行时 / rootfs 模板
│   ├── cgroupops/                # cgroup handler（v1）
│   ├── trace/ metrics/ util/
├── pkg/
│   ├── sandbox/                  # sandbox 元数据与 status 状态机（落盘）
│   ├── runtime/                  # runtime handler：runsc_handler + runsc client
│   ├── networkmanager/           # 网络：bridge（iptables NAT）唯一后端
│   ├── imagemanager/             # 镜像：oci / nydus / distillfs / S3
│   ├── volumemanager/ resourcemanager/ cgroupmanager/ store/ errord/
└── config/ configs/              # 配置
```

### 2.3 distill-fs（Rust）

一句话：**只读 FUSE 文件系统，对 raw 磁盘镜像 / Nydus RAFS 镜像做按需懒加载 + 4 MiB chunk 去重（LMDB）+ 可选 peer chunk 共享**。

- `Cargo.toml:7`：`"A read-only FUSE filesystem with lazy loading and chunk deduplication"`，keywords `["fuse","filesystem","nydus","deduplication","cache"]`。
- 依赖 `fuse-backend-rs`、`nydus-{api,rafs,storage,utils}`（`Cargo.toml:20-32`）。
- `README.md:32`：`The filesystem is read-only.`。模块 `backend / cli / fs / image / utils`（`src/lib.rs:15-22`）。
- 与 sandboxd 的关系是**单向**的：sandboxd 的 image-manager 把 distill-fs 当子进程拉起并挂载 rootfs（`akernel/src/sandboxd/README.md` 架构图；`builder/config/otel-collector-config.yaml:221` 注释 `distill_fs is still spawned as a subprocess by sandboxd's image-manager`）。distill-fs 源码里没有任何 `sandbox`/`sandboxd` 字样，不感知 sandbox 生命周期。
- 关键字核查：`checkpoint / resume / suspend / epoll / futex / model api / llm` **全部零命中**；`snapshot`（chunk-tracker 内存快照）、`proxy`（nydus `backend-http-proxy` 存储后端）、`stream`（tokio TcpStream）字面命中但语义无关。

**distill-fs 在用户描述的「stream 劫持 → checkpoint」链路中不扮演任何角色**，仅在 sandbox 启动前的 rootfs 准备阶段被动参与。

---

## 3. yuanrong 如何调度并启动 AKernel sandbox（用户说法验证：✅ 准确）

### 3.1 两级调度

yuanrong 的"调度"是分层的：

- **上层 / 节点实例级**：`functionsystem/functionsystem/src/function_master/global_scheduler/`（`global_sched.cpp` 用 `sched_tree.h` 构建调度树，`instance_manager/` 选实例，`scaler/` 弹性伸缩）。
- **下层 / 执行后端级**：`functionsystem/functionsystem/src/runtime_manager/manager/runtime_manager.cpp`，`RuntimeManager` actor 收 `"StartInstance"` 消息（`:59`），按 `EXECUTOR_TYPE` 用 `FindExecutor(...)` 分发到 `RuntimeExecutor / SupervisorExecutor / DockerExecutor / SandboxdExecutor`（`:512-532`）。

### 3.2 关键：CONTAINER → SANDBOXD 归一化

`runtime_manager.cpp:40-51`：

```cpp
// Container workloads are served exclusively by sandboxd. CONTAINER remains a
// wire-level compatibility value in existing requests and persisted responses,
// but it is normalized to SANDBOXD before executor lookup.
EXECUTOR_TYPE ContainerExecutorType() { return EXECUTOR_TYPE::SANDBOXD; }
EXECUTOR_TYPE NormalizeExecutorType(EXECUTOR_TYPE type) {
    return type == EXECUTOR_TYPE::CONTAINER ? ContainerExecutorType() : type;
}
```

容器型工作负载统一交给 `SandboxdExecutor`（`CreateSandboxdExecutor`，`:572-588`）。

### 3.3 yuanrong 是 client，sandboxd 是 server

`SandboxdExecutor`（`executor/sandboxd/sandboxd_executor.cpp`）通过 **UDS gRPC** 调用 sandboxd：

```cpp
// InitConfig（sandboxd_executor.cpp:283-303）
auto ep = litebus::os::GetEnv("CONTAINER_EP");     // UDS 路径来自环境变量
sandboxd_ = GrpcClient<runtime::v1::SandboxService>::CreateUdsGrpcClient(endpoint);
```

gRPC 封装（thin wrappers）：`DoStart→AsyncStart`、`DoDelete→AsyncDelete`、`DoRestore→AsyncRestore`、`DoRegister→AsyncRegister`、`DoWait→AsyncWait`、`DoList→AsyncList`、Stats→`AsyncStats`。

三条 Start 路径（`StartInstance`，`:317-374`）按 request 形态路由：
- **WarmUp**（`info.warmuptype() != NONE`）→ `StartWarmUp` → `Register` RPC（预注册可复用模板）。
- **Restore**（`!info.snapshotinfo().checkpointid().empty()`）→ `StartBySnapshot` → 下载 checkpoint、add ref、调 `Restore` RPC。
- **Normal** → `StartNormal` → `SandboxdRequestBuilder.Build()` 构造 `StartRequest` → `Start` RPC。

### 3.4 端到端调用链

```
用户/SDK
  │  (HTTP/WS, JWT)
  ▼
[frontend]  yuanrong/frontend/（Go, Gin）
  │  FaaS: /invocations → function_proxy
  │  Sandbox 直连: /sandbox/direct/* → httputil.ReverseProxy 反代到 sandbox 实例
  ▼
[function_proxy]  local_scheduler / request_router / traefik_route_cache
  ▼
[function_master: global_scheduler]  节点/实例选址 + scaler
  ▼
[runtime_manager: RuntimeManager]  StartInstance → NormalizeExecutorType(CONTAINER→SANDBOXD)
  ▼
[SandboxdExecutor]  StartNormal/StartWarmUp/StartBySnapshot → gRPC over UDS ($CONTAINER_EP)
  ▼
[sandboxd]  sandboxService.Start（internal/server/server.go）→ 准备 fs + resources
  ▼
[runsc / gVisor]  pkg/runtime/runsc_handler.go + runsc client → 真正拉起隔离 sandbox
```

旁路：`yuanrong/frontend/pkg/frontend/api/sandbox_direct.go:142` 的 `newSandboxDirectProxy` 把 `/sandbox/direct/*`、`/sandbox/tunnel/*` 反向代理到具体 sandbox 实例，**不经过 FaaS 调度**，用于已建好 sandbox 上的命令执行/端口转发。注意这是**出站反向代理（把外部请求送进 sandbox）**，不是劫持 sandbox 出向流量。

> 旁路中的 proxy 与用户说的"劫持 model API stream"无关：它代理的是用户→sandbox 方向，且只到 L4/L7 HTTP 转发，不做 TLS 解密。

---

## 4. 【核心】checkpoint / restore 的真相：proto 分叉

这是本次调研最重要的发现，直接关系到用户问题的回答。

### 4.1 两份 proto 的 RPC 集合不一致

| RPC | 开源 `akernel/.../sandbox-api.proto`（v0.1.0） | yuanrong 内部 `functionsystem/proto/posix/sandbox_api.proto` |
|---|---|---|
| Start | ✅ | ✅ |
| Delete | ✅ | ✅ |
| Wait | ✅ | ✅ |
| List | ✅ | ✅ |
| Stats | ✅ | ✅ |
| ListAvailableRuntimes | ✅ | ✅ |
| **Checkpoint** | ❌ 被裁掉 | ✅ |
| **Restore** | ❌ 被裁掉 | ✅ |
| **Register** | ❌ 被裁掉 | ✅ |
| **Unregister** | ❌ 被裁掉 | ✅ |
| **GetRegistered** | ❌ 被裁掉 | ✅ |

`diff` 结果（开源相对于 yuanrong 内部）缺失：`rpc Checkpoint / Restore / Register / Unregister / GetRegistered`。

开源 sandboxd 的 server 只注册 6-RPC 版 service：

```go
// akernel/src/sandboxd/internal/server/server.go:473
func (h *sandboxService) RegisterServer(server *grpc.Server) {
    runtime.RegisterSandboxServiceServer(server, h)
}
```

生成的 `sandbox-api_grpc.pb.go` 的 `SandboxServiceServer` 接口也只有这 6 个方法的 handler。也就是说：**yuanrong 的 `SandboxdExecutor` 调用 `Checkpoint/Restore/Register` 打到开源 sandboxd 上，会得到 gRPC "unimplemented" 错误**。开源 v0.1.0 不支持这些路径。

### 4.2 yuanrong 侧确实有进程级 checkpoint 编排

即使开源 sandboxd 裁掉了 RPC，yuanrong 侧的编排代码是真实存在的，能看清设计语义：

`yuanrong/.../executor/sandboxd/sandboxd_checkpoint_orchestrator.cpp` 的 `TakeSnapshot`：

```cpp
constexpr int64_t CHECKPOINT_TIMEOUT_NS = 30000000000;          // 30s
const std::string checkpointPath =
    fmt::format("/home/yuanrong/checkpoints/{}", checkpointID);

auto ckptReq = std::make_shared<runtime::v1::CheckpointRequest>();
ckptReq->set_id(sandboxID);
ckptReq->set_checkpoint_dir(checkpointPath);
ckptReq->set_timeout(CHECKPOINT_TIMEOUT_NS);
ckptReq->set_compress(true);
return DoCheckpoint(ckptReq).Then(... OnCheckpointDone ...);
```

- `DoCheckpoint` → `SandboxService::Stub::AsyncCheckpoint`。
- 成功后 `CkptFileManager::RegisterCheckpoint(checkpointID, checkpointPath, ..., ttl)` 把 checkpoint 目录注册到分布式存储（带 TTL）。
- `StartBySnapshot`（`sandboxd_executor.cpp:540-577`）走 `Restore` RPC：先下载 checkpoint、add ref，再 restore。
- `runtime_state_manager.cpp:69-144` 在 yuanrong 侧为每个 runtime 维护 `checkpointID` 元数据。
- 对外暴露为分布式原语：`yr.Group.suspend()`（保存实例 checkpoint）、`yr.Group.resume()`（读取并恢复），见 `yuanrong/docs/.../yr.Group.suspend.rst` / `resume.rst`。

**这是 CRIU/gVisor-save 风格的进程级 checkpoint/restore**——保存/恢复整个 sandbox 进程的执行状态，用于 idle parking、fork、迁移。它的触发源是**上层应用显式调用** `suspend/snapshot`，**不是**"检测到某条网络流就自动 checkpoint"。

### 4.3 开源 sandboxd 内"checkpoint"一词的真实含义

在开源 sandboxd Go 源码里 grep `checkpoint`，命中**全部**是元数据落盘语义，不是进程快照：

- `pkg/sandbox/status.go:82`：`// This field doesn't need to be checkpointed.`
- `pkg/sandbox/status.go:157,170,290,310`：`UpdateSync`/`LoadStatus` 把 `Status`（pid、时间戳、资源限额）JSON 序列化写到磁盘，失败信息 `failed to checkpoint status to %q`（`:310`）。
- `config/config.go:24`：注释 `(metadata checkpoint etc.)`。

`restore` 在开源 sandboxd 里指 `fsManager.Restore`（`internal/server/fsmanager.go:266-340`）：sandboxd 进程**重启后重建 rootfs/mount 引用**，不是恢复 sandbox 进程内存。

`Freeze/Thaw`：只在 `pkg/cgroupmanager/mock_test.go:44-45` 的**测试 mock** 里出现，`{ return nil }`；`CgroupHandler` 接口（`internal/cgroupops/cgroup.go:22-25`）只声明 `Create/Load`，**没有 Freeze/Thaw**；全仓库无调用方。cgroup v1 freezer 子系统从未初始化。`CRIU` 全仓库零命中。

> 结论：开源 sandboxd v0.1.0 既没有进程 checkpoint/restore（proto 裁掉、handler 不存在），也没有 cgroup freeze/thaw（接口未声明、未调用）。进程级 C/R 能力存在于 **yuanrong + 内部完整版 sandboxd** 链路里，开源版暂未开放。

---

## 5. 【核心】「model API stream 劫持」机制验证：❌ 完全不存在

针对用户转述的机制，逐条核查源码。

### 5.1 模型 API 感知：零命中

在整个 `akernel/src/sandboxd/`（Go）与 `akernel/src/distill-fs/`（Rust）源码中：

| 关键字 | 命中 |
|---|---|
| `anthropic` / `openai` / `claude` / `model api` / `llm` / `completion`（语义） | **0**（`completion` 仅 1 处 `EnableBashCompletion`，无关） |
| `apikey`（模型 API key 语义） | 0（仅 k8s service account token） |
| `stream 劫持` / `stream hijack` / `hijack stream` | 0 |

yuanrong 侧同样：`docs/source_zh_cn/.../data_stream/index.md:17` 明确写"数据流都是同步接口，没有类似 epoll 的多路复用能力"——即连数据流层面都没有 epoll 式多路复用。

**没有任何代码"知道"某条流是模型 API 流。**

### 5.2 网络劫持能力：只有 NAT，没有 L7/HTTPS 拦截

`sandboxd` 网络层只有唯一后端 `bridge`（iptables NAT），`config/constants.go:113` 注释 `// the only backend in v0.1.0`。`pkg/networkmanager/bridge/network.go` 只做两件事：

- SNAT/MASQUERADE（`:30-45`）：`-t nat -A POSTROUTING -s <range> -j MASQUERADE`，让 sandbox 出网。
- DNAT（`:76-99`）：`-t nat -A PREROUTING ... -j DNAT --to-destination <ip>:<port>`，**入向**端口转发（宿主机端口 → sandbox），服务于 SDK port forwarding。

关键字核查：`MITM / transparent / SO_ORIGINAL_DST / TPROXY / REDIRECT / eBPF / TC / clsact / bpf / nft` **全部 0 命中**。`go.mod` 唯一网络依赖是 `coreos/go-iptables v0.6.0`。源码里的 `http.Server` / `net.Listen` 全部用于 sandboxd 自身的 gRPC/UDS、imagemanager 的 unix 域 API、resourcemanager 的 metrics server——**没有一个监听在 sandbox 出向路径上**。

源码中 `proxy` 一词 100% 指两种东西：①拉容器镜像用的 HTTP forward proxy（`pkg/imagemanager/**` 的 `ProxyConfig.Url`）；②`resourcemanager/module.go:180` 注释里指上游调度代理。**没有面向 sandbox 出向流量的透明代理**。

关于 HTTPS：即使存在透明代理（目前并不存在），不做 TLS 解密/根 CA 植入就只能看到 SNI/IP/端口，看不到 body。源码里**没有任何 CA 证书注入 sandbox rootfs、没有任何 TLS 解密逻辑**（仅有的 `cert/TLS` 代码是拉镜像的 registry client 与 k8s client 的出站 client 侧配置）。

**结论：当前网络层无法截获 sandbox 内 claude 发往 `api.anthropic.com` 的 HTTPS body。**

### 5.3 freeze / proxy / 流识别的联动：不存在

- freeze 能力：开源版**不存在**（§4.3）。
- proxy/劫持能力：**不存在**（§5.2）。
- 流识别能力：**不存在**（§5.1）。
- 编排逻辑（"检测到 model API 流量 → freeze → 流完成 → thaw"）：**0 行代码**。

唯一带"流+暂停"语义的字段是 `pkg/runtime/runsc/network.go:101` 的 `PauseExternalNetworking`，但它是 gVisor netstack `createLinksAndRoutesArgs` 结构体的一个字段（gVisor save/restore 场景暂停外部网络用），**在 sandboxd 里没有任何 setter 或调用路径**，是死字段。

### 5.4 epoll / futex 在源码里的真实含义

| 关键字 | 源码命中 | 含义 |
|---|---|---|
| `epoll` | `akernel/src/sandboxd/pkg/cgroupmanager/cgroup_oom.go:59,65` | 注释提到未来可用 inotify+epoll 读 OOM eventfd，替代每 watcher 占一个 OS 线程。**与"等模型流"无关** |
| `futex` | yuanrong 仅 `datasystem/.skills/ds-pr-review/SKILL.md:216` 等 PR 审查清单字样 | review 指南，**非运行时代码** |

源码里**没有任何"用 epoll/futex 等待 model API 返回再唤醒 sandbox"的实现**。

---

## 6. 开发者文档对相关概念的描述（设计意图 vs 已实现）

### 6.1 checkpoint/restore：论文核心规划，但 v0.1.0 明确不含

`akernel/README.md:35-39`：

```
- **40 ms cold start\***: Fork-based launch with lazy loading ...
- **Sandbox isolation**: gVisor today, with Kata Containers support\* planned
- **Checkpoint/Restore\***: Save and restore sandbox state for fast recovery
\* Planned for an open-source release and not available in AKernel v0.1.0.
```

`akernel/README.md:165-171` Roadmap：`Fork-based sandbox launch` / `Kata Containers` / `Sandbox checkpoint and restore` / `GKE and AWS` / `Cgroup v2 node support` —— **全部未实现**。

`akernel/AGENTS.md`：`AKernel v0.1.0 node runtime currently requires Linux x86-64 nodes with cgroup v1.`（不支持 cgroup v2，而 v2 freezer 才是"freeze 整个 cgroup"的干净路径）。`akernel/src/sandboxd/README.md:72-75` 的 Known limitations：`Only gVisor, cgroup v1, netstack sandbox networking, and iptables are supported in v0.1.0.`

### 6.2 论文里的 checkpoint-on-idle：触发源是"等待"，不是"劫持"

这是理解用户问题最关键的一节。开发者**确实**规划了"Agent 等待时 checkpoint 释放资源"，但其语义与"劫持 model API stream"**根本不同**。

`AKernel_Docs/Paper_outline.md:403-405`（K3. Checkpoint-on-idle）：

> 当 Agent 等待 LLM、外部 API、人审或长时间 I/O 时，AKernel 将 process hibernate 或 checkpoint，并释放 CPU/memory。对于并行探索，AKernel 从同一 checkpoint fork 多个 branch。

`AKernel_Docs/AKernel_Paper_Structure.md:134`：

> Agent 经常等待 LLM、外部 API、人审、网络 | checkpoint-on-idle 能释放资源

`AKernel_Docs/Introduction_outline.md:104-111`（设计三：生命周期感知调度）：

> 活跃环境继续运行；等待模型或外部事件的环境可以释放部分资源；长时间空闲环境自动检查点并回收计算资源；后续请求到来时快速恢复。

`AKernel_Docs/AKernel_Paper_Structure.md:377-384` 生命周期状态机：`Created → Starting → Running → Idle → Checkpointed`，分支 `Forked / Hibernated / Restoring`。

**关键辨析**：
- 开发者对 LLM 的处理方式是 **"Agent 在等待 LLM 响应期间，由 AKernel 主动 checkpoint-on-idle 释放 CPU/memory"**——触发者是**调度器/生命周期管理器对"空闲/等待"状态的判断**，手段是 **hibernate/checkpoint 整个 process**，目的是**释放资源**。
- 用户转述的机制是 **"劫持 model API 的网络 stream，stream 活动期间 freeze sandbox，stream 完成后 resume"**——触发者是**对流量的观察**，手段是 **freeze（不一定 checkpoint 落盘）**，目的是**在 stream 进行时省资源**。

两者表象相似（都是"等模型时把 sandbox 摘掉"），但**实现路径完全不同**：前者是 OS 级生命周期调度（不需要看网络流），后者是网络中间人劫持（需要解密 HTTPS、识别模型流）。**论文 outline 描述的是前者；用户听到的"stream 劫持"更像是把前者通俗化/具体化时的口误或过度演绎，并不是论文里写明的机制，更不是源码里实现的机制。**

### 6.3 作者自己建议删掉"等模型=epoll"这类类比

`AKernel_Docs/Introduction_outline.md:146-148`：

> 建议删除或谨慎使用：
> ...
> "等待模型回复等于 epoll"；
> "扩缩容等于处理器热插拔"；
> 暂无数据支撑的"分钟级"和"毫秒级"结论。

**作者明确不建议**把"等模型回复等于 epoll"作为论据。`futex` 在全部 markdown 中零命中；`epoll` 仅此一处且被建议删除。`Introduction_outline.md:78-83` 还明确标注 `spawn / fork / exec` 当前"含义存在混用，正式写作前必须明确"。

### 6.4 model api / llm proxy / stream 劫持：全文档零命中

在 `akernel/` + `AKernel_Docs/` 全部 `.md` 中，`model api / llm proxy / stream 劫持 / stream hijack / hijack.*stream` **零命中**。文档里出现 `LLM` 时一律表述为"Agent 等待 LLM"（把 LLM 当外部等待源），从不表述为"劫持/代理 LLM 调用"。

### 6.5 suspend 不是 AKernel 自己的术语

`suspend` 命中 12 个文件但**全部在 Surveys/（外部调研）**；核心 README/outline 用 `hibernate / pause / checkpoint-on-idle / 挂起`。`yr.Group.suspend/resume` 是 yuanrong Group 编程模型的原语，属于 §4.2 描述的进程 C/R 对外暴露。

---

## 7. 这样做（如果真做）有什么好处 —— 机制价值分析

尽管当前未实现，用户问"这么做有什么好处"值得认真回答。把"Agent 等待 LLM 时把 sandbox 摘掉"这件事，按两种实现路径分别分析利弊。

### 7.1 路径 A：OS 级 checkpoint-on-idle（论文规划的方向）

做法：调度器/生命周期管理器判断 sandbox 进入"等待 LLM/外部 API/人审"状态，对该 sandbox 做 hibernate/checkpoint（释放 CPU、可释放内存），事件到来时 restore。

好处：
1. **资源回收**：Agent session 中 LLM thinking 常占 60%–90% 时间（`Paper_outline.md:66` 列为待测 TODO）。这段时间 sandbox 完全 idle 却占着 vCPU/RAM。checkpoint 后资源可给别的 Agent，显著提高节点密度、降低 cost/resource-hours。
2. **长会话可行性**：把"一个 Agent = 一个常驻 sandbox"变成"一个 Agent = 一串 checkpoint lineage"，使数小时/数天的 session 在经济上可行。
3. **fork/branch 友好**：从 checkpoint fork 出多个 branch 做并行探索、RL rollout，是 planning primitive（`Paper_outline.md:343,405`）。
4. **不需要看网络流**：触发依据是"进程是否在等待"（可由 SDK 显式 yield、或 runtime 观察到阻塞），无需破坏 HTTPS，对模型供应商透明，不引入 TLS 解密的安全/合规问题。

代价：checkpoint/restore 本身有成本（pause 写内存层、restore demand-fault），需要 working-set 感知来决定是否划算；释放内存会丢 page cache 热度。需要一套 idle 检测 + checkpoint 调度策略（什么时候 check、check 后留多少 resident、何时 restore）。

> 这一路的价值是真实的，也是 AKernel 论文 K3 的卖点。但它**不依赖** stream 劫持。

### 7.2 路径 B：model API stream 劫持（用户转述的方向）

做法：透明代理 + 根 CA 植入 + TLS 解密，识别 sandbox 内发往模型 API 的 HTTPS 流；在 stream 活动期间 freeze sandbox；收到完整响应后 resume。

理论上的"好处"：
1. **自动、无需 Agent 配合**：不需要 SDK 显式 yield，任何用 claude code/openai SDK 的程序都自动获得"等待时摘机"。
2. **精确到 stream 粒度**：能区分"正在收 LLM token"和"在跑本地代码"，比粗粒度 idle 检测更准。

但代价极大、且与论文方向相悖：
1. **HTTPS 劫持需在 sandbox rootfs 植入私有根 CA**，否则看不到 body；这与 sandbox"安全隔离、策略受控"的定位冲突，多租户下是合规与安全噩梦。
2. **gVisor netstack + 透明代理**实现复杂：要么 TPROXY/eBPF 重定向出向流量，要么改 runsc netstack 钩子，再叠加 TLS MITM。当前代码零基础。
3. **freeze ≠ checkpoint**：若只是 cgroup freezer 冻结进程，内存仍占着，对"释放内存"这个最大收益无帮助；若要真省内存还得落盘 checkpoint，那就回到路径 A，劫持 stream 这一步是多余的。
4. **stream 活动期间 freeze sandbox 会破坏 LLM client 的 HTTP 连接语义**：客户端通常在一个 HTTP 连接上 SSE 读 token，freeze 进程意味着 socket 停摆，proxy 必须自己缓冲整个响应再回放——等于把 SSE 流式降级成批量返回，**牺牲了流式 token 的用户体验（首 token 延迟、可中断）**，与"等完整结果再 resume"的描述一致，但这恰恰是 LLM 应用不想要的。
5. **作者本人建议删掉"等模型=epoll"类比**（§6.3），说明这条具体化路径不在论文论据里。

> 综合判断：路径 B 的"自动性"看似诱人，但把它作为 checkpoint 的触发条件是**本末倒置**——真正有价值的是"等待期释放资源"（路径 A 已覆盖），而 stream 劫持引入的安全、合规、复杂度、流式语义破坏等成本远超其"自动检测"的边际收益。AKernel 论文选择路径 A 是更合理的设计。

### 7.3 一个可能澄清：用户描述也许是对路径 A 的通俗化

"stream 被劫持 → checkpoint → 结果返回后 resume" 很可能是一句口语化的转述，其实想表达的是路径 A：**"当 claude code 在等模型 API 返回时（stream 在途），AKernel 把这个 sandbox checkpoint 掉；模型结果回来后，再 resume sandbox 继续执行。"** 这句话描述的**外部行为**（等模型时摘机、回来后继续）与论文 checkpoint-on-idle 完全一致；只是"劫持 stream"这个**实现手段**的猜测不准确——AKernel 不需要劫持 stream，它靠的是生命周期调度（sandbox 进入等待态就 checkpoint，靠 SDK yield 或 idle 检测或应用层告知）。建议向开发者确认是不是这个意思。

---

## 8. 若要落地"等待 LLM 时 checkpoint-on-idle"，当前代码缺什么

以路径 A（论文方向）为准，开源 v0.1.0 距离落地需要的最小增量：

1. **进程 checkpoint/restore**：恢复 sandboxd proto 里被裁掉的 `Checkpoint/Restore` RPC 及 handler；接 gVisor save/restore 或 CRIU。内部版已有 `SandboxdCheckpointOrchestrator` 编排逻辑可参考（§4.2）。
2. **cgroup v2 支持 + freezer**：v0.1.0 只支持 cgroup v1（`AGENTS.md`），而 v2 才有干净的 `cgroup.freeze` 一次性冻结整个 cgroup 的能力；`CgroupHandler` 需补 `Freeze/Thaw`。
3. **idle / 等待态检测**：sandbox 状态机从 `{Running,Exited,Unknown}` 扩展到含 `Idle/Checkpointed/Hibernated`（论文 `AKernel_Paper_Structure.md:377-384` 已设计）；需要一个信号源告诉 sandboxd"这个 sandbox 正在等 LLM"——最干净的方式是 **SDK/agent 显式 yield**（`akernel_sdk` 增加一个"我即将等待外部事件，可 checkpoint"的调用），而非观察网络流。
4. **restore locality 调度**：`Paper_outline.md:409` 要求根据 checkpoint/workspace diff/package cache/repo/browser profile/region 选恢复节点——需要 object plane + checkpoint repository（可借 AgentENV 式 OverlayBD layer chain 或 Agent Substrate 式 snapshot CR）。
5. **crash consistency / commit protocol**：checkpoint 落盘、catalog、binding、logical Agent identity 之间的 prepare/commit 协议（开源版目前完全没有这层）。
6. **策略层**：何时 checkpoint（idle 阈值、TTL）、checkpoint 后留多少 resident、何时 evict——避免 thrashing。

若执意走路径 B（stream 劫持），额外还需要：透明代理（TPROXY 或 runsc netstack 钩子）+ 根 CA 植入 rootfs + TLS 解密 + 模型流识别 + freeze/stream/thaw 编排状态机。**不推荐**，理由见 §7.2。

---

## 9. 与既有调研的关系

本文与既有 surveys 互补：
- `20260810T143748Z-agentenv-checkpoint-restore-...md` 详述了 AgentENV / Agent Substrate 如何做进程 C/R 与 layered state data plane。AKernel 的 checkpoint-on-idle 若要落地，state data plane 可借鉴那条路径；但 AKernel 的**触发模型**（Agent 生命周期感知）是 AgentENV/Substrate 当前都不充分的——这正是 AKernel 的论文贡献点（`Paper_outline.md:303` 两级调度 + `:403` checkpoint-on-idle）。
- 本文首次厘清「开源 sandboxd 裁剪了 Checkpoint/Restore RPC」「yuanrong 内部 proto 保留它们」这一**开源 vs 内部分叉**，这对论文写作时陈述"已实现/正在实现/计划实现"的边界至关重要（`Paper_outline.md:633` 明确要求划清这条边界）。

---

## 10. 最终判断

1. **yuanrong 调用 sandboxd 启动 AKernel**：✅ 准确。`RuntimeManager → SandboxdExecutor → gRPC(Start) → sandboxd → runsc`。
2. **"实现了 epoll/futex 语义、劫持 model API stream、stream 期间 checkpoint、结果回来 resume"**：❌ 当前开源源码无任何实现，文档亦无描述。最接近的真相是论文规划的 **checkpoint-on-idle**（Agent 等待 LLM 时由生命周期调度器主动 checkpoint），其**外部行为**与转述相似，但**实现手段不是 stream 劫持**，且开源 v0.1.0 连 Checkpoint/Restore RPC 都已裁掉。
3. **proto 分叉**：开源 sandboxd 仅 6 个 RPC；yuanrong 内部 proto 有 11 个（多 Checkpoint/Restore/Register/Unregister/GetRegistered）。yuanrong 的 checkpoint 编排器（`SandboxdCheckpointOrchestrator`）是面向内部完整版写的，调到开源 sandboxd 会 unimplemented。
4. **机制价值**：「等待 LLM 时摘机」本身有价值（释放资源、长会话、fork），但这由 OS 级 checkpoint-on-idle 实现即可，不需要、也不应该靠 stream 劫持。建议向开发者确认：转述的"stream 劫持 + checkpoint"是否其实是对 checkpoint-on-idle 的口语化表达。

---

## 11. 源码证据索引

### 11.1 yuanrong 调度链路
- [CONTAINER→SANDBOXD 归一化](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/manager/runtime_manager.cpp:40)
- [StartInstance executor 分发](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/manager/runtime_manager.cpp:59)
- [SandboxdExecutor UDS 连接（CONTAINER_EP）](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:283)
- [三条 Start 路由](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:317)
- [DoStart→AsyncStart](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:1227)
- [yuanrong 内部 proto（11 RPC）](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/proto/posix/sandbox_api.proto:22)
- [sandbox 直连反代（出站，非劫持）](/mnt/u/hukeyang/AKernel/yuanrong/frontend/pkg/frontend/api/sandbox_direct.go:142)
- [global_scheduler](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/function_master/global_scheduler/global_sched.cpp:23)

### 11.2 sandboxd（开源 v0.1.0）
- [开源 proto（6 RPC，无 Checkpoint/Restore）](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:22)
- [只注册 6-RPC service](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/server.go:473)
- [runsc handler：仅 Create/Start/Wait/Delete/ListJSON](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/runsc_handler.go:58)
- [状态机无 PAUSED](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/sandbox/status.go:33)
- ["checkpoint"=status 落盘](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/sandbox/status.go:157)
- [Freeze/Thaw 仅 mock_test](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/cgroupmanager/mock_test.go:44)
- [CgroupHandler 无 Freeze/Thaw](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/cgroupops/cgroup.go:22)
- [网络唯一后端 bridge/iptables](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/networkmanager/bridge/network.go:30)
- [iptables 仅 SNAT+DNAT](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/networkmanager/bridge/network.go:76)
- [epoll 仅 OOM 监控注释](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/cgroupmanager/cgroup_oom.go:59)
- [v0.1.0 限制：gVisor/cgroup v1/netstack/iptables](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/README.md:72)

### 11.3 checkpoint（yuanrong 内部版）
- [TakeSnapshot→Checkpoint RPC（30s/compress）](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_checkpoint_orchestrator.cpp:71)
- [StartBySnapshot→Restore RPC](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:540)
- [runtime checkpoint 元数据](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/runtime_state_manager.cpp:69)
- [yr.Group.suspend/resume](/mnt/u/hukeyang/AKernel/yuanrong/docs/source_zh_cn/invocation_and_deployment/Python/yr.Group.suspend.rst)

### 11.4 设计意图（论文/docs）
- [v0.1.0 不含 Checkpoint/Restore](/mnt/u/hukeyang/AKernel/akernel/README.md:37)
- [Roadmap：fork/kata/C&R/cgroup v2](/mnt/u/hukeyang/AKernel/akernel/README.md:165)
- [checkpoint-on-idle 规划](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:403)
- [Agent Process 状态机](/mnt/u/hukeyang/AKernel/AKernel_Docs/AKernel_Paper_Structure.md:377)
- [建议删"等模型=epoll"类比](/mnt/u/hukeyang/AKernel/AKernel_Docs/Introduction_outline.md:146)
- [spawn/fork/exec 语义待统一](/mnt/u/hukeyang/AKernel/AKernel_Docs/Introduction_outline.md:78)
- [两级调度 + locality](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:303)
- [openYuanrong + AFaaS 定位](/mnt/u/hukeyang/AKernel/AKernel_Docs/outline_draft.md:25)

### 11.5 distill-fs
- [职责：只读 FUSE + 懒加载 + 去重](/mnt/u/hukeyang/AKernel/akernel/src/distill-fs/Cargo.toml:7)
- [sandboxd 把 distill-fs 当子进程拉起（注释）](/mnt/u/hukeyang/AKernel/akernel/builder/config/otel-collector-config.yaml:221)
- [distill-fs 在 image plane（非 checkpoint 链路）](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:427)
