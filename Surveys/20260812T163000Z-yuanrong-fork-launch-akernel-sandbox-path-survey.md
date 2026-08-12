# yuanrong 基于 fork 启动 AKernel sandbox 的路径调研

> 调研时间：2026-08-12（Asia/Shanghai）
> 文档时间戳：`20260812T163000Z`
> 调研对象：`yuanrong/`（openYuanrong，functionsystem C++ / frontend Go / api Python）、`akernel/src/sandboxd/`（开源 v0.1.0）、`AKernel_Docs/`
> 证据边界：本文以本地源码静态审计为"当前实现"证据，以项目 README/docs/outline 为"作者声明或设计意图"证据。未在目标硬件上运行，所有功能判定基于源码，不涉及性能复现。
> 关联文档：[20260812T160000Z 调度链路与 stream 劫持调研](./20260812T160000Z-akernel-yuanrong-source-structure-stream-hijack-checkpoint-survey.md)（本文是其在"fork 启动"方向上的深入）。

---

## 0. 一句话结论

开源 AKernel v0.1.0 当前**只能走 "Normal Start"**（从 rootfs 用 gVisor runsc 全新起一个沙箱），**没有任何 fork 启动能力**。

"基于 fork 启动 sandbox"在 **yuanrong 内部版**里由两步协同实现：
1. **WarmUp**：`SandboxdExecutor::StartWarmUp` 调 sandboxd 的 `Register` RPC，注册一个可复用的 `SandboxTemplate`，并（当 `warmup=seed` 时）置 `make_seed=true`，要求 sandboxd 预创建一个 **seed sandbox**（被派生的母本）。
2. **派生**：后续普通 `Start` 请求若带 `template_id`（且与已注册模板兼容），sandboxd 服务端就从对应 seed **clone/fork** 出新沙箱。

但**真正的 fork 执行逻辑在 sandboxd 服务端，未开源**。开源 sandboxd 的 proto 把 `Register / Unregister / GetRegistered / Checkpoint / Restore` 这 5 个 RPC 全部裁掉（只剩 6 个 RPC），`template_id` 字段虽保留但被**显式忽略**，runsc adapter 也只调用 `create / delete / list` 三个子命令。因此 yuanrong 的 WarmUp/Restore 路径打到开源 sandboxd 上会得到 gRPC `Unimplemented` 错误且**无 fallback**。

文档层面：`akernel/README.md` 的 "40 ms cold start, Fork-based launch" 与 "Checkpoint/Restore" 均标星号 **Planned / not available in v0.1.0**；论文 outline 里 fork 还承担"Agent Process 并行探索生命周期原语"（Fork-Explore-Commit）的更上层语义。

---

## 1. 调研范围与证据等级

### 1.1 主要源码范围

| 主题 | 入口 |
|---|---|
| 三条 Start 路径路由 | `yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp` |
| StartNormal 的 template_id 复用 | `.../executor/sandboxd/sandboxd_request_builder.cpp` |
| WarmupType 枚举与触发 | `.../common/metadata/metadata_type.h`、`.../function_proxy/local_scheduler/instance_control/instance_ctrl_actor.cpp` |
| 内部版 proto（11 RPC） | `yuanrong/functionsystem/proto/posix/sandbox_api.proto` |
| 开源 sandboxd proto（6 RPC） | `akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto` |
| 开源 sandboxd server | `akernel/src/sandboxd/internal/server/server.go` |
| 开源 runsc adapter | `akernel/src/sandboxd/pkg/runtime/runsc/client.go`、`rpc.go`、`network.go`、`runsc_handler.go` |
| 用户可见配置 | `yuanrong/api/python/yr/cli/scripts.py`、`.../common/metadata/metadata.cpp` |
| 文档/论文定位 | `akernel/README.md`、`AKernel_Docs/{Paper_outline,AKernel_Paper_Structure,Introduction_outline,outline_draft}.md` |

### 1.2 证据等级

| 等级 | 含义 | 本文用法 |
|---|---|---|
| A | 固定源码/测试直接支持 | "已实现（开源）""未实现（源码无命中或显式注释）""proto 分叉" |
| B | 项目 docs/README/outline 声明 | "设计意图""路线图""论文规划" |
| C | 推演、需进一步验证 | "fork 执行逻辑应在 sandboxd 服务端（未开源）" |

---

## 2. 概念澄清：源码里有三种 "fork"，必须分开

这是本文最容易踩坑的地方。术语 "fork" 在 AKernel/yuanrong 体系里至少有三层不同含义，混用会得出错误结论：

| 层次 | 含义 | 出处 | 开源状态 |
|---|---|---|---|
| **L1. AFaaS fork 启动** | 从预热的 seed/golden checkpoint 快速派生新沙箱，达到极低冷启动延迟（README 的 "40ms cold start, Fork-based launch"） | `akernel/README.md:35`；`AKernel_Docs/AKernel.md` 引用 AFaaS "Fork in the Road" | 未开源，v0.1.0 不可用 |
| **L2. Agent Process fork 原语** | 从一个 running/checkpointed Agent 派生多个 branch 做并行探索（Fork-Explore-Commit），用于 RL rollout、candidate patch 评估 | `Paper_outline.md:343,405,572` | 论文规划，未实现 |
| **L3. gVisor runsc fork** | runsc/nanovisor 层的 clone/save/restore，是 L1 的底层载体 | `akernel/README.md:167` Roadmap "Fork-based sandbox launch based on gVisor" | 未实现（runsc 只调 create/delete/list） |

此外还有两个**容易误判为 fork**的东西，需要排除：

- **OS 级 `fork()`+`exec()`**：`RuntimeExecutor`（非 SandboxdExecutor）用 `litebus::Exec` 起用户语言 runtime 进程。这是**启动用户进程**，不是"从已有 sandbox fork 出新 sandbox"，与本文主题无关。
- **yuanrong 的 `make_seed` / `SandboxTemplate`**：这是 L1 在 **client 侧**的实现（注册 seed + 带 template_id 请求派生），真正的 fork 执行在 sandboxd 服务端。

> 本文聚焦 **L1 的 client 侧实现路径**（即 yuanrong 怎么发起 fork 启动），并指出其服务端依赖未开源。L2 属于论文规划的生命周期原语，在 §6 讨论。

`Introduction_outline.md:83` 作者自己也标注："目前草稿中 spawn、fork、exec 的含义存在混用，正式写作前必须明确。"——这条调研正是为厘清边界。

---

## 3. 三条 Start 路径全景

`SandboxdExecutor::StartInstance`（`sandboxd_executor.cpp:317-374`）按 request 形态路由到三条路径：

```
StartInstance(request)
  │
  ├─ info.warmuptype() != NONE ?
  │     YES → StartWarmUp  → Register RPC   （注册可复用 seed/template）
  │
  ├─ info.snapshotinfo().checkpointid() 非空 ?
  │     YES → StartBySnapshot → Restore RPC （从 checkpoint 恢复）
  │
  └─ 否则 → StartNormal → Start RPC          （全新创建，可带 template_id 复用 seed）
```

源码（`:333, :358-369`）：

```cpp
const bool isWarmUp = info.warmuptype() != static_cast<int32_t>(WarmupType::NONE);
...
if (isWarmUp) {
    future = StartWarmUp(request, cmdArgs, port, envs, guard);
} else if (!info.snapshotinfo().checkpointid().empty()) {
    future = StartBySnapshot(context);
} else {
    future = StartNormal(context);
}
```

| 路径 | 触发条件 | 调用的 sandboxd RPC | 语义 | 开源 sandboxd 可跑？ |
|---|---|---|---|---|
| **WarmUp** | `warmuptype ∈ {SEED, PRELOAD}` | `Register` | 预注册可复用模板/seed | ❌（开源无 Register RPC） |
| **Restore** | `checkpointid != ""` | `Restore` | 从 checkpoint 恢复沙箱 | ❌（开源无 Restore RPC） |
| **Normal** | 其余 | `Start` | 全新创建；命中已注册模板时带 `template_id` 派生 | ✅（但 `template_id` 被忽略） |

---

## 4. fork 启动路径（核心）：WarmUp 注册 seed + Normal 带 template_id 派生

"基于 fork 启动 sandbox"**不是独立的第四条路径**，而是**横跨 WarmUp 和 Normal 两条路径的协同**。下面分两步拆解。

### 4.1 第一步：WarmUp 注册一个 seed/template（`StartWarmUp`）

`sandboxd_executor.cpp:418-462`（注释原文："Warm-up path (Register a reusable SandboxTemplate)"）：

```cpp
auto registerReq = std::make_shared<runtime::v1::RegisterRequest>();
auto *tmpl = registerReq->add_templates();
tmpl->set_id(runtimeID);
tmpl->set_runtime(info.container().runtime());
*tmpl->mutable_rootfs() = info.container().rootfsconfig();
tmpl->set_make_seed(info.warmuptype() == static_cast<int32_t>(WarmupType::SEED));   // :433
...                                                   // 填 command / envs / mounts
return DoRegister(registerReq)                       // → SandboxService::Stub::AsyncRegister
    .Then(... &SandboxdExecutor::OnWarmUpRegistered ...);
```

- `DoRegister`（`:1269-1284`）通过 UDS gRPC 调用 sandboxd 的 `Register` RPC。
- `OnWarmUpRegistered`（`:464-483`）：注册成功后把 `runtimeID` 插入 `warmupRuntimes_` 和 `registeredTemplateIDs_`（`:475-476`）。
- `YR_SEED_FILE` 环境变量（`:451-452`）：若设置，注入到 template envs，供 sandbox 内 runtime 阻塞等待 seed 就绪（`yuanrong/api/python/yr/main/yr_runtime_main.py:143-146`）。
- 拆除走 `Unregister` RPC（`UnregisterWarmUp`，`:745-763`），不走 Delete。

#### 4.1.1 WarmupType 枚举

`functionsystem/src/common/metadata/metadata_type.h:255-273`：

```cpp
enum class WarmupType {
    NONE     = 0,
    SEED     = 1,   // make_seed=true，要求 sandboxd 预创建 seed sandbox（fork 母本）
    PRELOAD  = 2,
    INVALID  = 255,
};
// 字符串映射："seed"→SEED, "preload"→PRELOAD, "none"→NONE
```

#### 4.1.2 SandboxTemplate proto 定义（内部版）

`yuanrong/functionsystem/proto/posix/sandbox_api.proto:281-298`：

```protobuf
message SandboxTemplate {
  string id = 1;
  string runtime = 2;
  RootfsConfig rootfs = 3;
  bool make_seed = 4;   // "asks sandboxd to pre-create a seed sandbox when supported"
  repeated string command = 5;
  map<string,string> envs = 6;
  string cwd = 7;
  repeated Mount mounts = 8;
}
message RegisterRequest { repeated SandboxTemplate templates = 1; }
```

**`make_seed=true` 是 fork 启动的关键**：seed sandbox 就是被派生的母本，sandboxd 服务端预创建它之后，后续带 `template_id` 的 Start 才能从它快速 clone/fork。

#### 4.1.3 触发点：函数注册时的 FunctionWarmUp

`functionsystem/src/function_proxy/local_scheduler/instance_control/instance_ctrl_actor.cpp:7225-7242`：函数注册时若 `warmup != NONE/INVALID`，发 `DeployInstanceRequest`（带 `set_warmuptype`），经 `functionAgentMgr_->RegisterToWarmUp` 落到 `SandboxdExecutor::StartWarmUp`。

#### 4.1.4 用户可见配置入口

- 函数 metadata 的 `"warmup"` 字段：`metadata.cpp:553-554`、`service_json.cpp:563-564`。
- **Python CLI 默认填 `"warmup": "seed"`**：`yuanrong/api/python/yr/cli/scripts.py:2032`（部署函数时）。
- **akernel SDK 侧完全不暴露 warmup/template/fork**：`akernel/sdk/python` 的 `Sandbox` API 只有 `cpu/memory/run/exec/文件/PTY`（见 `akernel/CLAUDE.md` SDK 示例）。

> 即：fork 启动在 yuanrong 函数模型里是**对函数部署透明**的（函数 metadata 配 `warmup: seed` 即可），而 AKernel 的 `akernel_sdk.Sandbox` 直连路径**完全没有这个能力**。

### 4.2 第二步：Normal Start 带 template_id 触发派生

`sandboxd_request_builder.cpp:439-445`：

```cpp
// Reuse the registered baseline template only when that final result still
// matches it; never mutate a registered template with a request overlay.
const auto &templateID = info.container().id();
if (params.registeredTemplateIDs.count(templateID) > 0 && IsTemplateCompatible(info, *start)) {
    start->set_template_id(templateID);   // 在 StartRequest 上带 template_id
}
```

- 只有当 `container.id()` 命中 `registeredTemplateIDs_`（即存在已注册 seed）**且** `IsTemplateCompatible`（请求与模板兼容，不会用 overlay 改写模板）时，才在 `StartRequest` 上设 `template_id`。
- `StartRequest.template_id` 的 proto 注释（内部版 `sandbox_api.proto:186-187`）原文：

  > `// TemplateID is the registered SandboxTemplate ID to start from. Only requests with this field set are eligible to reuse a registered seed/fork path.`

  **这是 "fork path" 在 yuanrong 侧最直接的文字证据**：`template_id` 命中即走 seed/fork path。

- 真正的"从 seed clone/fork 出新沙箱"发生在 **sandboxd 服务端**（收到带 `template_id` 的 Start 时执行），这部分逻辑**未开源**（§5）。
- `registeredTemplateIDs_` 还在 Restore 路径（`:532`）和 Sync（重连时调 `GetRegistered` RPC 同步，`:1205, :1315-1321`）中使用。

### 4.3 fork 启动的完整时序（内部版）

```
[函数注册, warmup=seed]
        │
        ▼
instance_ctrl_actor: FunctionWarmUp
        │  DeployInstanceRequest(warmuptype=SEED)
        ▼
SandboxdExecutor::StartWarmUp
        │  RegisterRequest{ SandboxTemplate{ id, runtime, rootfs, make_seed=true, ... } }
        │  ──gRPC/UDS──▶ sandboxd.Register
        ▼
sandboxd（内部版）: 预创建 seed sandbox（= fork 母本），记录 template id
        │
   ...(后续请求)...
        │
        ▼
SandboxdExecutor::StartNormal  (warmuptype=NONE, checkpointid 空)
        │  builder.Build(): 命中 registeredTemplateIDs_ 且兼容 → set_template_id(tid)
        │  StartRequest{ template_id=tid, ... }
        │  ──gRPC/UDS──▶ sandboxd.Start
        ▼
sandboxd（内部版）: 检测到 template_id → 从 seed clone/fork 出新沙箱（快速启动）
```

---

## 5. 为什么开源 v0.1.0 跑不了 fork：proto 分叉 + 服务端缺失

### 5.1 开源 proto 裁掉了 5 个 RPC

| RPC | 开源 `akernel/.../sandbox-api.proto` | 内部 `yuanrong/.../sandbox_api.proto` |
|---|---|---|
| Start / Delete / Wait / List / Stats / ListAvailableRuntimes | ✅ | ✅ |
| **Register** | ❌ | ✅ |
| **Unregister** | ❌ | ✅ |
| **GetRegistered** | ❌ | ✅ |
| **Checkpoint** | ❌ | ✅ |
| **Restore** | ❌ | ✅ |

### 5.2 开源 sandboxd server 未实现这些方法，且继承 Unimplemented 哨兵

`akernel/src/sandboxd/internal/server/server.go`：

- `sandboxService` 实现的方法仅 `Start(:816) / Delete(:749) / Wait(:1000) / List(:278) / Stats(:311) / ListAvailableRuntimes(:360)`。
- `:92, :649` 嵌入：

  ```go
  UnimplementedSandboxServiceServer: runtime.UnimplementedSandboxServiceServer{},
  ```

  gRPC 的 `UnimplementedXxxServer` 默认实现会让所有未覆盖方法返回 `codes.Unimplemented`。因此 yuanrong 调 `Register/Restore/GetRegistered` 会直接拿到 Unimplemented 错误。

### 5.3 开源 `template_id` 字段被显式忽略

`akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:91-94`：

```proto
// TemplateID is reserved for API compatibility with downstream builds that
// can start from pre-registered sandbox templates. The open-source runtime
// path does not implement template reuse and ignores this field.
string template_id = 2;
```

**开源 sandboxd 即使收到带 `template_id` 的 Start 也忽略它**，不会触发 fork。这是 fork 在开源版"字段保留但语义不实现"的硬证据。

> 注意：开源 sandboxd 的 `internal/langrtmanager/manager.go:25` 有一个 `SeedInfo{Cid, Cmd, Envs}` 结构体，但它只是 langruntime 记录用的元数据，**与 yuanrong 的 `make_seed` 不是一回事**，不要混淆。

### 5.4 开源 runsc adapter 只调 create/delete/list

`akernel/src/sandboxd/pkg/runtime/runsc/client.go` 调用的 runsc 命令/RPC 全清单：

| 操作 | 调用方式 | 位置 |
|---|---|---|
| `runsc ... create` | 二进制 exec | `client.go:107-139` |
| `runsc ... delete [--force]` | 二进制 exec | `client.go:280-299` |
| `runsc ... list --format json` | 二进制 exec | `client.go:305-319` |
| `containerManager.CreateLinksAndRoutes` | urpc（control socket） | `client.go:205` |
| `containerManager.StartRoot` | urpc | `client.go:208` |
| `containerManager.Wait` | urpc | `client.go:262` |

`runscClient` 接口（`runsc_handler.go:58-64`）只有 `Create/Start/Wait/Delete/ListJSON`。**没有 `runsc save/restore/checkpoint/pause/resume`**，也没有 gofer sapi 或 container clone 调用。gVisor 本身的 fork 能力完全未被使用。

### 5.5 预留但未接线的死字段：PauseExternalNetworking / AllowConnectedOnSave

`akernel/src/sandboxd/pkg/runtime/runsc/network.go:101-102`：

```go
type createLinksAndRoutesArgs struct {
    ...
    PauseExternalNetworking bool   // :101  gVisor save/restore 期间冻结外部网络
    AllowConnectedOnSave    bool   // :102  字面即 "save" 场景
}
```

这两个字段是 gVisor `containerManager.CreateLinksAndRoutes` RPC 的原生字段（从 gVisor 源码拷贝以保持控制面 ABI 对齐），**只在 save/restore（checkpoint/fork）场景才用**。但在 `BuildNetworkArgs`（`network.go:156-222`）里**它们从未被赋值**——构造的 args 只填了 LoopbackLinks/FDBasedLinks/Defaultv4Gateway。这是 fork/checkpoint 在开源代码里"预留位但未接线"的证据。

### 5.6 yuanrong 侧无 fallback

`SandboxdExecutor::DoRegister`（`:1269-1284`）等 wrapper 直接把 gRPC 失败映射成 `NormalResponse{success=false}`；`OnWarmUpRegistered`（`:470-473`）看到 `!response.success()` 直接返回 `RUNTIME_MANAGER_WARMUP_FAILURE`；`DoRestore` 同理。**没有任何"降级到 StartNormal"的逻辑**。开源组合下 WarmUp/Restore 会直接失败。

---

## 6. 论文里 fork 的定位（L2：Agent Process 生命周期原语）

除 L1（AFaaS fork 启动）外，论文还把 fork 提升为 **Agent Process 的生命周期原语**，语义更重。

`AKernel_Docs/Paper_outline.md`：

- `:170`："AKernel 要把 `checkpoint`、`restore`、`fork`、`hibernate`、`migrate`、`commit`、`abort` 变成 Agent Process 生命周期原语。"
- `:233` 生命周期：`create → useful_start → run → checkpoint/hibernate/fork/migrate → restore → commit/abort → destroy`。
- `:313` Sandbox Daemon 要"支持 create、fork、restore、hibernate、migrate、commit、destroy"。
- `:343`："**fork 支持并行探索和 RL rollout**。"
- `:405`："对于并行探索，AKernel 从同一 checkpoint fork 多个 branch，最后 commit 成功分支或 abort 失败分支。"（**Fork-Explore-Commit 模式**）
- `:547` 评估指标："fork/branch latency"。
- `:572-573` Experiment E6："评估 parallel exploration、RL rollout、candidate patch evaluation 中 fork/commit/abort 的延迟和状态复用收益。"

`AKernel_Docs/AKernel_Paper_Structure.md`：`:283` stateful lifecycle、`:380` 状态机含 `Forked/Hibernated/Restoring`、`:542` "sandbox clone/fork"。

`AKernel_Docs/AKernel.md:7,16`：论文行文对标 SigmaOS / YuanRong / **AFaaS（"Fork in the Road"）**。

**L1 与 L2 的关系**：L1（seed-fork 快速启动）提供启动性能；L2（Agent fork 多 branch）是上层语义，实现上很可能复用 checkpoint/restore（先 checkpoint 一个 running sandbox，再多次 Restore 成多个 branch）或复用 seed-fork。论文承认（`Introduction_outline.md:83`）spawn/fork/exec 含义"目前草稿中混用，正式写作前必须明确"。

---

## 7. 文档声明 vs 已实现：汇总表

| 能力 | 开源 v0.1.0 | 内部版 | 文档声明 | 关键证据 |
|---|---|---|---|---|
| Normal Start（全新沙箱，runsc create） | ✅ | ✅ | — | `server.go:816`、proto 6 RPC、`runsc/client.go:107` |
| WarmUp / Register seed（fork 母本注册） | ❌ Unimplemented | ✅ | — | `server.go` 无 Register；内部 proto :36-41,281-298；`sandboxd_executor.cpp:418-462` |
| Normal Start 带 `template_id` 派生（fork） | ❌ 字段被忽略 | ✅ | README "Fork-based launch\*" Planned | 开源 proto :91-94 注释；内部 proto :186-187 "seed/fork path"；`sandboxd_request_builder.cpp:439-445` |
| Restore from checkpoint | ❌ Unimplemented | ✅ | README "Checkpoint/Restore\*" Planned | `server.go` 无 Restore；内部 `sandboxd_executor.cpp:487-577` |
| runsc save/restore/checkpoint 子命令 | ❌ 未调用 | n/a | — | `runsc/client.go` 仅 create/delete/list + 3 urpc |
| PauseExternalNetworking / AllowConnectedOnSave | ❌ 死字段 | n/a | — | `runsc/network.go:101-102` 未赋值 |
| OS 级 fork() 起用户进程 | n/a（RuntimeExecutor 才有） | ✅ RuntimeExecutor | — | `runtime_executor.h`（与 sandbox fork 无关） |
| AFaaS 40ms fork 启动 | ❌ | 未开源 | README :35,39 | "Planned... not available in v0.1.0" |
| Agent fork/branch（Fork-Explore-Commit） | ❌ | 未开源 | Paper_outline :343,405,572 | 论文 roadmap |
| `make_seed` / `YR_SEED_FILE` seed 机制 | ❌（服务端缺失） | ✅ | — | `sandboxd_executor.cpp:433,451`；`yr_runtime_main.py:143-146` |
| 用户可见 warmup 配置 | ❌ akernel SDK 无 | ✅ yuanrong 函数 metadata `"warmup":"seed"` | — | `scripts.py:2032`；`metadata_type.h:255-273` |

---

## 8. 端到端：开源用户当前能体验到的 vs 文档承诺的

| 用户入口 | 开源 v0.1.0 实际行为 | 文档承诺 |
|---|---|---|
| `akernel_sdk.Sandbox(cpu=..., memory=...)` | Normal Start：runsc 从 rootfs 全新起沙箱，无 fork、无 seed | README 宣传的 "40ms cold start, Fork-based launch" |
| yuanrong 函数部署 `warmup=seed` | WarmUp→Register RPC 打到开源 sandboxd → gRPC Unimplemented → `WARMUP_FAILURE` | 内部版下应预创建 seed，后续派生 |
| yuanrong 从 snapshot 恢复 | Restore RPC 打到开源 sandboxd → Unimplemented → 失败 | README "Checkpoint/Restore" Planned |
| Agent fork 多 branch（RL rollout） | 无任何代码路径 | Paper_outline Fork-Explore-Commit |

**即：开源用户当前拿到的只是"Normal Start 起一个 gVisor 沙箱"，README/论文里的 fork-based launch / 40ms cold start / checkpoint-restore / Agent fork 全部不在开源 v0.1.0 里。** 性能声明 `40 ms cold start*` 的星号脚注 `* Planned ... not available in AKernel v0.1.0` 必须在论文写作时如实标注（`Paper_outline.md:633` 已要求划清"已实现/正在实现/计划实现"边界）。

---

## 9. 若要落地 fork 启动，当前代码缺什么

以 L1（seed-fork 快速启动）为准，开源版需要的最小增量：

1. **sandboxd proto 恢复 `Register/Unregister/GetRegistered`**（client 侧 yuanrong 已就绪，`SandboxdExecutor::StartWarmUp` / `DoRegister` / `Sync` 全套都在），并实现服务端 handler：接收 `SandboxTemplate`，按 `make_seed` 预创建 seed sandbox。
2. **sandboxd Start handler 不再忽略 `template_id`**：当请求带 `template_id` 且命中已注册 seed 时，走 clone/fork 分支而非全新 create。
3. **gVisor 层 fork 执行**：runsc adapter 增加 `save/restore` 或 container clone 调用（当前只 create/delete/list）；接通 `PauseExternalNetworking/AllowConnectedOnSave`（目前死字段）。或走 AFaaS 的 nanovisor fork（未开源）。
4. **seed 生命周期管理**：seed 的引用计数、兼容性校验（`IsTemplateCompatible` yuanrong 侧已有）、销毁（`Unregister`）。
5. **L2 Agent fork**：若要支持 Fork-Explore-Commit，还需 checkpoint/restore（`Checkpoint/Restore` RPC）+ branch lineage + commit/abort 协议。

> **建议**：开源 v0.1.0 的 RunscServiceHandler（`runsc_handler.go`）当前结构清晰，添加 fork 的最小改动点是 ① proto 加回 Register/Restore/Checkpoint；② runscClient 接口与 client.go 增加 save/restore；③ StartSandbox 分叉 template_id 路径。AFaaS nanovisor 的 fork 具体实现细节需参考 AFaaS 论文（`AKernel_Docs` 下 `References/Chai et al_2025_Fork in the Road.pdf`）。

---

## 10. 最终判断

1. **"基于 fork 启动 sandbox"在 yuanrong 侧是真实设计的**：WarmUp 调 `Register` 注册 `SandboxTemplate`（`make_seed=true` 预创建母本），后续 Normal Start 带 `template_id` 触发 sandboxd 从 seed clone/fork。这是 client 侧完整闭环。
2. **但 fork 的执行逻辑在 sandboxd 服务端，未开源**。开源 sandboxd 裁掉了 `Register/Unregister/GetRegistered/Checkpoint/Restore` 5 个 RPC，`template_id` 被显式忽略，runsc adapter 只调 create/delete/list。开源组合下 fork 启动**完全不可用**，且无 fallback。
3. **开源 v0.1.0 当前唯一能跑的是 Normal Start**（从 rootfs 全新起 gVisor runsc 沙箱）。
4. **文档层面**：README 的 "40ms fork-based launch" 与 "Checkpoint/Restore" 标星 Planned；论文把 fork 进一步提升为 Agent Process 生命周期原语（Fork-Explore-Commit，用于并行探索/RL）。spawn/fork/exec 的精确语义作者标注"混用、待统一"。
5. **论文写作注意**：必须如实区分"AFaaS 已验证的 fork 启动（未开源，性能数字引自 AFaaS）" vs "AKernel 开源版未实现" vs "Agent fork 是规划原语"。不能把 README 的 40ms 当作 AKernel 开源版已测得的结果。

---

## 11. 源码证据索引

### 11.1 三条 Start 路径
- [StartInstance 路由](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:317)
- [WarmUp 分支判断](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:333)
- [StartBySnapshot→Restore](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:487)
- [DoRestore→AsyncRestore](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:1334)

### 11.2 WarmUp / Register / seed
- [StartWarmUp 构造 RegisterRequest](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:418)
- [make_seed = (warmuptype==SEED)](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:433)
- [YR_SEED_FILE 注入](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:451)
- [OnWarmUpRegistered 记录 registeredTemplateIDs_](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:464)
- [DoRegister→AsyncRegister](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:1269)
- [UnregisterWarmUp→Unregister RPC](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:745)
- [Sync/GetRegistered 重连同步](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_executor.cpp:1205)

### 11.3 template_id 派生（fork client 侧触发）
- [builder 命中模板时 set_template_id](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/runtime_manager/executor/sandboxd/sandboxd_request_builder.cpp:439)
- [StartRequest.template_id 注释（seed/fork path）](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/proto/posix/sandbox_api.proto:186)

### 11.4 WarmupType 与触发
- [WarmupType 枚举 NONE/SEED/PRELOAD/INVALID](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/common/metadata/metadata_type.h:255)
- [FunctionWarmUp 触发点](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/function_proxy/local_scheduler/instance_control/instance_ctrl_actor.cpp:7225)
- [Python CLI 默认 warmup=seed](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/cli/scripts.py:2032)
- [YR_SEED_FILE 运行时阻塞](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/main/yr_runtime_main.py:143)

### 11.5 内部版 proto（11 RPC）
- [SandboxTemplate / RegisterRequest](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/proto/posix/sandbox_api.proto:281)
- [Register/Unregister/GetRegistered RPC](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/proto/posix/sandbox_api.proto:36)

### 11.6 开源 sandboxd 裁剪证据
- [开源 proto 仅 6 RPC](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:22)
- [开源 template_id 被显式忽略](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:91)
- [server 嵌入 UnimplementedSandboxServiceServer](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/server.go:92)
- [server 仅实现 6 方法（Start 在 :816）](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/server/server.go:816)
- [runsc 仅调 create/delete/list](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/runsc/client.go:107)
- [runscClient 接口 5 方法](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/runsc_handler.go:58)
- [PauseExternalNetworking/AllowConnectedOnSave 死字段](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/pkg/runtime/runsc/network.go:101)

### 11.7 文档/论文定位
- [40ms fork-based launch Planned](/mnt/u/hukeyang/AKernel/akernel/README.md:35)
- [Roadmap: Fork-based sandbox launch based on gVisor](/mnt/u/hukeyang/AKernel/akernel/README.md:167)
- [AFaaS = nanovisor](/mnt/u/hukeyang/AKernel/AKernel_Docs/outline_draft.md:30)
- [fork 为 Agent Process 生命周期原语](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:170)
- [Fork-Explore-Commit](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:405)
- [spawn/fork/exec 语义待统一](/mnt/u/hukeyang/AKernel/AKernel_Docs/Introduction_outline.md:83)
- [对标 AFaaS Fork in the Road](/mnt/u/hukeyang/AKernel/AKernel_Docs/AKernel.md:7)
