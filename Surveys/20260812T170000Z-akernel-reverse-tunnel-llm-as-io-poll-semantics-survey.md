# ReverseTunnel 语义与「LLM 调用即 I/O / poll 调度」可行性分析

> 调研时间：2026-08-12（Asia/Shanghai）
> 文档时间戳：`20260812T170000Z`
> 调研对象：`akernel/sdk/python/`（HttpReverseTunnel）、`yuanrong/api/python/yr/sandbox/`（TunnelClient/TunnelServer）、`yuanrong/api/rust/rrt-daemon/`（生产实现 rrt-runtime）、`yuanrong/frontend/`、`akernel/src/sandboxd/`（资源/调度能力边界）
> 证据边界：源码静态审计，未运行。性能/延迟数字不作为实测结论。
> 关联：[20260812T160000Z 调度链路与 stream 劫持](./20260812T160000Z-akernel-yuanrong-source-structure-stream-hijack-checkpoint-survey.md)、[20260812T163000Z fork 启动路径](./20260812T163000Z-yuanrong-fork-launch-akernel-sandbox-path-survey.md)。本文是它们在"可观测 I/O 模型"方向的延伸。

---

## 0. 结论先行

你的直觉**方向正确，且 ReverseTunnel 恰好提供了实现它所需的"可观测锚点"**，但当前代码只走到了前半程——"父亲能看见 child 的 LLM 调用"成立，"父亲据此把 child 调度走"在开源 v0.1.0 还差关键的最后一公里。具体：

1. **ReverseTunnel 的真实语义**：它**不是**"父亲劫持 child 出向流量"，而是**"把父亲侧（SDK 控制进程）的 HTTP/HTTPS 服务暴露给 child sandbox 调用"**。数据流方向是 child → 父亲。child 在 sandbox 内访问 `http://127.0.0.1:{listen_port}`，请求经一条反向 WebSocket（穿 Traefik）送到父亲进程内的 TunnelClient，由父亲用 httpx 打到 `target`。TLS 在父亲侧终止，child 只看到明文 loopback HTTP。

2. **"父亲知晓 child 对 LLM 的调用语义"——成立**。因为数据必然流经父亲进程，父亲手里有每条请求的完整明文（method/path/headers/body + 响应），且每个请求有独立 id（multi-plex over 单 WS），父亲能区分并发、能测往返时延、能 hold/改写/拒绝。这正是把"LLM 调用"建模成一次"可观测 I/O"所需的全部信息。

3. **"把这次 I/O 当 poll 事件、child 等待期被调度走"——半成立**。ReverseTunnel 给出了"child 正在等 I/O"的清晰信号（请求发出、响应未回）。但开源 v0.1.0 缺：
   - **观测面缺口**：TunnelClient 没有暴露任何 request 级 hook/metrics/callback，父亲进程内的数据"看得见但没接口拿"。
   - **控制面缺口**：sandbox 没有 pause/resize/checkpoint 的运行时 RPC（开源 sandboxd 只有 Start/Delete/Wait/List/Stats），cgroup 只到 v1 且未接 freezer。即便父亲知道"该把 child 调度走了"，也无处下手。
   - **执行面缺口**：Agent 也没有"我正在等外部 I/O，可被调度"的显式 yield。

4. **与 K8s / SigmaOS 的差别**：你的判断准确——K8s 的调度粒度是 Pod 级、秒~分钟级、对"容器内进程在等 LLM"完全无感；SigmaOS 有 process-level 的 I/O 感知和 spark/fork 但不面向 LLM I/O 语义。"把 LLM 调用变成一等公民的、可被调度器观测的 I/O 事件"确实是 AKernel 作为 cloud-native OS 最有差异化的论点之一。ReverseTunnel 是实现这个论点的天然底座，但需要补齐观测/控制/执行三层。

---

## 1. 调研范围与证据等级

| 等级 | 含义 | 用法 |
|---|---|---|
| A | 固定源码直接支持 | "已实现""未实现（无命中或显式注释）" |
| B | docs/README/示例声明 | "设计意图""示例演示" |
| C | 推演 / 建议补齐 | "若要落地需新增 X" |

主要源码入口见 §10 证据索引。

---

## 2. 先纠正一个关键误解：ReverseTunnel 的方向

**用户原始设想**（需修正）："child sandbox 走这个 ReverseTunnel 去访问 LLM 服务" + "father sandbox 劫持 child 对 LLM 的调用"。

实际源码语义（A 级证据）：

- `akernel/sdk/python/akernel_sdk/types.py:158-164`，`HttpReverseTunnel` docstring：
  > "Expose an SDK-side HTTP or HTTPS service **inside a sandbox**. `reverse_port` carries the WebSocket tunnel through the AKernel gateway. Sandbox applications call `url`, which points at the loopback HTTP listener on `listen_port`."

- `akernel/sdk/python/README.md:200-229`：
  > "A reverse tunnel lets **sandbox code** call an HTTP or HTTPS service **reachable from the machine running the SDK** ... `listen_port` is the loopback HTTP listener used **inside the sandbox** ... `sandbox.reverse_tunnel.url` is always `http://127.0.0.1:<listen_port>` ... For an HTTPS target, **the SDK-side tunnel client performs the TLS handshake** and certificate verification. The sandbox application talks only to its loopback HTTP listener."

**方向是 child → 父亲**（"reverse" 指数据流方向反向于"服务暴露"的常规方向：通常是把远端服务暴露给本地，这里是把本地/SDK 侧服务暴露给远端 sandbox）。

数据流：

```
child sandbox 内进程
    │  curl http://127.0.0.1:{listen_port}/v1/chat/...     （明文 HTTP，loopback）
    ▼
[sandbox 内 rrt-runtime]  127.0.0.1:{listen_port} = Port B (HTTP listener)
    │  帧化为 {id, method, path, headers, body}（JSON+base64）
    │  经单条 WebSocket（Port A, 0.0.0.0:{reverse_port}）
    ▼
[Traefik 公网入口]  PathPrefix(/{safeID}/{reverse_port}) → 节点 host:port
    │  WebSocket 透传
    ▼
[父亲进程（SDK 控制进程）内]  TunnelClient（daemon 线程）
    │  httpx.AsyncClient 打到 target
    │  ── 若 target=https://api.openai.com：TLS 在此终止/发起 ──
    │  ── 若 target=http://127.0.0.1:local_port：父亲本地 HTTP server ──
    ▼
target（真实 LLM API 或父亲本地 proxy）
    │  response 原路返回：父亲 → WS → sandbox Port B → child 进程 recv
```

关键澄清两点：

- **不是"劫持"**：child 并没有"试图直连 LLM 然后被中间人截获"。是 child **主动**访问自己的 loopback 地址（`127.0.0.1:listen_port`），由 sandbox 内 rrt-runtime 把这个请求经隧道送给父亲。child 从头到尾不知道 LLM 在哪。这是一种**显式的、应用层的服务注入**，不是透明劫持。
- **"father sandbox"在代码里 = SDK 控制进程**：akernel SDK 没有任何"sandbox 里再 spawn sandbox / father-child sandbox 嵌套"的 API（`sandbox.py` 的 `Sandbox` 是顶层对象，由控制进程持有）。所以你说的"father sandbox"在当前代码里对应的就是**跑 `akernel_sdk`、持有 `Sandbox(handle)` 的那个控制进程**。这不影响你的论证（控制进程当然可以是另一个 sandbox 里的 agent 进程，只是当前 SDK 不把它表达成"father sandbox"概念）。

> 本文下文统一用"**控制端**"指代父亲侧（持有 Sandbox 的进程，可能在本地、也可能是上层 agent sandbox），用"**sandbox**"指代被 spawn 的 child。

---

## 3. ReverseTunnel 端到端实现（A 级证据摘要）

### 3.1 三个角色

| 角色 | 进程位置 | 实现 |
|---|---|---|
| **TunnelServer** | sandbox 内（rrt-runtime 进程） | Rust `rrt-daemon/src/runtime/tunnel.rs`（生产）/ Python `yr/sandbox/tunnel_server.py`（后备） |
| **WebSocket 透传** | Traefik + docker port-forward | Traefik 规则由 `traefik_registry.cpp:99-123` 写 etcd |
| **TunnelClient** | 控制端进程内（daemon 线程） | Python `yr/sandbox/tunnel_client.py` |

### 3.2 端口

| 名字 | 默认值 | sandbox 内绑定 | 角色 |
|---|---|---|---|
| `reverse_port`（Port A / ws_port） | 8765 | `0.0.0.0:8765` | WebSocket 端，控制端反向连入 |
| `listen_port`（Port B / http_port） | 8766 | `127.0.0.1:8766` | sandbox 内进程访问的 HTTP listener |

`tunnel.rs:201-221` 同时绑这两个端口；`handler.go:828-855 prepareSandboxTunnel` 注入 `RRT_TUNNEL_WS_PORT=8765` / `RRT_TUNNEL_HTTP_PORT=8766` 并下发 port-forward。

### 3.3 协议：multi-plex over 单 WebSocket

- 物理层：**单条 WebSocket**（控制端一个 `_current_ws`，sandbox 一个 `_sdk_ws`）。
- 逻辑层：**每请求一个 id**（sandbox 侧 `make_id()` 生成 `fid`），帧 `{id, method, path, headers, body}` 在同一 WS 上交错传输，按 id demux。
- 并发：控制端 `asyncio.Semaphore(max_http_concurrency)`，默认 10，可由 `YR_TUNNEL_HTTP_CONCURRENCY` 配置。
- body：**整段缓冲后 base64**（非流式）；上限 `YR_TUNNEL_MAX_BODY_SIZE`（256MB）/ Rust 64MB。

### 3.4 同步语义与超时（对"调度走"至关重要）

- sandbox 侧发出 `http_req` 后 `await wait_for(fut, timeout=600s)`（`HTTP_TIMEOUT=600s`）。在这 600s 内，sandbox 里发起这次 HTTP 调用的进程**阻塞在 recv 上**（典型 `urllib.request.urlopen` / `requests.get`）。
- 控制端 hold 响应 → child block；600s 后 sandbox 侧返回 `504 Tunnel timeout`。
- WS 断开时 in-flight 请求缓存 120s，重连后重发。

> **这条同步语义是整个"poll 调度"论点的物理基础**：child 在等 LLM 时，进程本来就阻塞在 socket recv 上、不占 CPU——这和传统 Linux 进程 `read()` 一个慢设备时被移出 runqueue 是**同一个现象**。ReverseTunnel 让这个"慢设备"（LLM）的入口出现在控制端，于是控制端有了观测和控制它的可能。

---

## 4. 可行性分析：把 LLM 调用建模为"可观测 I/O"

你的论证链条是：

> child 等 LLM = 一次 I/O 访问 → 控制端据此实现 poll → child 在等待期被"调度走"（缩资源 / checkpoint）。

逐环评估。

### 4.1 环节①：控制端能观测 child 的 LLM 调用语义？—— ✅ 成立（数据可见）

控制端 `TunnelClient._handle_http`（`tunnel_client.py:475-541`）手里有**完整明文**：

- 请求：`frame.method / frame.path / frame.header_items / frame.body`；
- 响应：`resp.status_code / resp.headers / response_body`；
- TLS 已在控制端终止（`httpx.create_ssl_context`，`:273-275`），body 是明文。

这足以让控制端理解"语义"：比如识别 `/v1/chat/completions`、读出 `model` 字段、判断这是 LLM 调用而非普通 fetch。每个请求有独立 `fid`，并发可区分。往返 wall-clock 在控制端可测。

**但**（评级 B/C）：当前代码**没有暴露任何接口让你拿到这些数据**——
- `TunnelClient.__init__` 没有 interceptor / callback / middleware 参数；
- 只有连接级 `logger.info`，没有 request 级日志、没有 metrics、没有 OpenTelemetry；
- `HttpReqFrame` 不携带 sandbox 侧的请求进入时间戳，所以控制端只能测"往返总时延"，测不出"纯 LLM 处理时延"。

> 结论：**数据天然可观测，但 API 缺失**。要把"观测"变成可用能力，最小改动是在 `_handle_http` 加一个 `on_request(frame) / on_response(frame, latency)` 钩子（C 级建议）。

### 4.2 环节②：把观测变成 poll 事件？—— ✅ 概念成立，需自行编排

poll 的语义是：多个 I/O 等待中，谁 ready 了就唤醒谁处理。映射到这里：

- 一个控制端可能同时管多个 child sandbox（每个一个 `Sandbox` handle），每个 child 可能有多个 in-flight LLM 请求。
- 控制端拿到"child A 的请求 fid=3 在等 LLM、child B 的 fid=7 已返回"这类事件后，可以建一个事件循环（`select`/`epoll` 风格）做调度决策。

ReverseTunnel 的 multi-plex + per-request id 天然支持这种编排：每个 in-flight 请求就是一个"未 ready 的 fd"，响应到达就是一个"ready 事件"。控制端是这些事件的唯一汇聚点（因为所有 child 的 LLM 流量都经它）。

**但**：当前 SDK 没有提供"多个 Sandbox 统一事件循环"的抽象。`Sandbox` 是同步阻塞 API（`commands.run` 阻塞到完成）。要做 poll，控制端需要：
- 异步化的 SDK（或直接在控制端 asyncio 里跑多个 TunnelClient task）；
- 一个集中器把 N 个 child 的 tunnel 事件汇总成一个可 `await` 的事件流。

这是工程量不大但当前不存在的部分（C 级）。

### 4.3 环节③：child 在等待期被"调度走"——缩资源 / checkpoint？—— ❌ 开源 v0.1.0 缺失执行面

这是整个论点最关键、也最薄弱的一环。即便控制端通过 poll 知道"child A 现在在等 LLM，可以摘掉"，**它对 child A 能施加什么调度动作？**

逐个核对开源 sandboxd（`akernel/src/sandboxd/`）的能力：

| 想施加的动作 | 开源 v0.1.0 是否可做 | 证据 |
|---|---|---|
| 通知 child"让出 CPU"（类 `sched_yield`） | ❌ 无此 RPC/信号 | sandbox API 仅 Start/Delete/Wait/List/Stats |
| 冻结 child 进程（cgroup freezer，类 `SIGSTOP` 但不杀） | ❌ cgroup v1 freezer 未接；`CgroupHandler` 无 Freeze/Thaw | `internal/cgroupops/cgroup.go:22`；`mock_test.go:44` |
| 缩小 child 资源（降 CPU/memory limit） | ⚠️ 部分：cgroup `Update` 可改 cpu/mem limit，但 sandbox 无运行时 resize RPC | `runsc_handler.go:66 updateCgroup` 仅在 Start 时调；无 Resize RPC |
| checkpoint child 并释放资源（hibernate） | ❌ Checkpoint/Restore RPC 被裁；runsc 无 save/restore | 见 [fork 启动调研](./20260812T163000Z-yuanrong-fork-launch-akernel-sandbox-path-survey.md) §5 |
| 直接 kill + 之后重建 | ✅ Delete 可做，但等价于"杀进程"，非"调度走" | — |
| 什么都不做，仅靠 child 自己阻塞在 recv | ✅ child 等 LLM 时本来就不占 CPU（自然 sleep） | §3.4 同步语义 |

最后一行其实是个**很重要的正向发现**：**即使没有任何调度动作，child 在等 LLM 的整个期间，其进程本来就阻塞在 tunnel 的 `await wait_for` 上、不占 CPU**。也就是说，"调度走、不占 CPU 资源"这个目标，在 **CPU 维度上 ReverseTunnel 已经自动达成了**——这是应用层同步等待带来的天然副作用，和内核把 `read()` 慢设备的进程移出 runqueue 等价。

真正没解决的是 **memory 维度**：child 阻塞时它的 RSS（page cache、运行时堆、已加载的模型/索引）仍占着内存。要释放内存，就得 checkpoint（落盘后回收）或 migrate——而这正是开源版完全没有的部分。

> 结论：**"CPU 维度的调度走"已隐式实现（child 等待时天然不占 CPU）；"memory 维度的调度走"（缩资源/checkpoint）在开源 v0.1.0 无执行路径。**

### 4.4 综合可行性矩阵

| 论点环节 | 数据/机制是否存在 | API 是否就绪 | 评级 |
|---|---|---|---|
| child 的 LLM 调用流经控制端 | ✅ ReverseTunnel 天然如此 | — | A |
| 控制端看到完整调用语义（明文） | ✅ TLS 在控制端终止 | ❌ 无 hook/metrics | B（数据）/ C（API） |
| 控制端区分并发、测时延 | ✅ per-request id，multi-plex | ❌ 无 timestamp 字段 | B |
| 控制端 hold/改写/拒绝请求 | ✅ 控制端是同步转发方 | ❌ 无 interceptor | C |
| 把多个 child 的等待建成 poll 事件循环 | ✅ 概念成立 | ❌ SDK 同步、无集中器 | C |
| child 等 LLM 时不占 CPU | ✅ 天然阻塞在 recv | — | A |
| child 缩资源（运行时 resize） | ⚠️ cgroup Update 存在 | ❌ 无 resize RPC | C |
| child checkpoint/释放内存 | ❌ Checkpoint RPC 被裁 | ❌ | 未实现 |
| child 显式 yield（"我可被调度"） | ❌ 无此原语 | ❌ | 未实现 |

---

## 5. 为什么这是 AKernel 相对 K8s / SigmaOS 的核心差异（论点支撑）

你的判断：这是 AKernel 作为 cloud-native OS 相比传统 K8s 或 SigmaOS 最大的差别。从源码看，这个论点**站得住，但要精确表述**。

### 5.1 vs Kubernetes

K8s 的调度边界是 Pod：kube-scheduler 做 bin-pack（秒~分钟级），kubelet 管 cgroup 静态限额。Pod 内进程"在等 LLM"对调度器**完全不透明**——K8s 看不见进程级 I/O 等待，更看不见"这条 HTTPS 流是 LLM 调用"。所以 K8s 无法在 Agent 等 LLM 时自动回收资源，只能靠 HPA/空闲超时缩容整个 Pod（粗、慢、有冷启动代价）。

ReverseTunnel 的差异在于：它把 child 的（被建模为 I/O 的）LLM 调用**显式地引到控制端**，使调度器第一次能"看见"这次 I/O 的语义和时序。这是把调度粒度从 Pod 级下沉到"进程的 I/O 等待"级的前提。

### 5.2 vs SigmaOS

SigmaOS 的 procs 有 spark/fork、I/O 感知调度、`was` 让出语义——它**确实**把进程级 I/O 等待纳入调度，是离 AKernel 最近的前作。但 SigmaOS 的 I/O 是**传统 I/O**（文件系统 / 网络 RPC），它的调度器优化的是"proc 在等慢 I/O 时不占机器"。AKernel 的差异化论点应该是：

- **把 LLM 调用提升为一等公民的、有语义的 I/O 事件**：不只是"进程在等网络"，而是"agent 在等一个会产出 tool-call 的推理"，调度器可以据此做更聪明的决策（预取、批量、fork 探索、优先级）。
- **LLM I/O 的时间尺度特殊**：单次 LLM 调用 1~60s，远长于传统 RPC，又远短于 Pod 缩容周期。这个尺度恰好落在 K8s（太粗）和 OS 内核调度（太细、单机）之间的空档，需要一个"云 OS"级的调度器来管。ReverseTunnel 让这个尺度的 I/O 变成可观测、可调度的事件。

### 5.3 论点的精确表述（建议用于论文）

> AKernel 把 Agent 对 LLM 的调用从"不透明的出向 HTTPS"重构为"经 ReverseTunnel 显式汇入控制平面的、带语义的 I/O 事件"。这使得以 LLM-I/O 等待为核心信号的生命周期感知调度（缩资源 / checkpoint / fork）成为可能——这是 Pod 级的 Kubernetes 和 proc 级的 SigmaOS 都没有覆盖的中间尺度，是 AKernel 作为 agent cloud OS 的核心特征。

⚠️ **务必同步的限定**（否则会被 reviewer 攻击）：开源 v0.1.0 只具备"观测数据流经控制端"和"CPU 自然让出"这两个前置条件；**调度执行面（resize/checkpoint/fork/yield）尚未实现**。论文应把它作为 **design + 部分实现 + roadmap**，而非"已奏效的系统"。

---

## 6. 两条落地路径

把这套机制真正跑起来，有两条路径，工程量与优雅度不同。

### 路径 A：target = 控制端本地 LLM proxy（推荐，零侵入隧道层）

控制端在本地起一个 HTTP server（FastAPI / aiohttp），把 `HttpReverseTunnel(target="http://127.0.0.1:{local}")` 指向它。child 的 LLM 请求直接进入控制端自己的 server handler。

- 观测：handler 里想记什么记什么（任意语义解析、metrics、trace）。
- 控制：handler 里想延迟/拒绝/缓存/改写随便做（天然单进程内代码）。
- 调度决策：handler 收到请求 → 通知调度器"child X 进入 LLM 等待" → 调度器对 child 施加动作 → 收到 LLM 响应 → 唤醒。
- **优点**：完全不碰 TunnelClient 源码，示例（`examples/reverse_tunnel.py`）已演示此模式。
- **缺点**：只覆盖"走 proxy 的 LLM 调用"；若 child 里跑了 claude code 直连 `api.anthropic.com`，你得让它的 base_url 指向 loopback（需配置/拦截）。

### 路径 B：在 TunnelClient 内建观测/控制 hook

改 `yr/sandbox/tunnel_client.py`：给 `TunnelClient.__init__` 加 `on_request`/`on_response` 回调，或加一个 `Interceptor` 抽象。这样无论 target 是本地 proxy 还是真实 API，所有隧道流量都可观测/可控。

- **优点**：覆盖所有经隧道的流量，统一。
- **缺点**：侵入 yuanrong 运行时代码；且只对"经隧道的流量"有效（child 若有第二条直连出口仍看不到——但当前 sandbox 网络经 iptables NAT 出网，那条路是存在的，只是没被观测）。

> **对 AKernel 的建议**：路径 A 适合快速验证和论文 prototype（工作量最小、语义最干净）；路径 B 适合做成平台级能力。两者都依赖 §7 的执行面补齐。

### 关键约束：无论哪条路径，都受 600s 超时和"非流式 body"限制

- `HTTP_TIMEOUT=600s`：LLM 调用若被控制端 hold 超过 600s，child 会收到 504。做 checkpoint+restore 时必须保证 restore 在 600s 内完成，或显式延长该超时。
- body 整段缓冲：流式 SSE 会被降级为"攒完整段再回放"（[stream 劫持调研](./20260812T160000Z-akernel-yuanrong-source-structure-stream-hijack-checkpoint-survey.md) §7.2 也指出过这点）。这对 LLM 流式输出体验有损，需权衡。

---

## 7. 要把"poll 调度"跑通，开源版还缺什么（按层）

1. **观测层**：TunnelClient 加 request/response hook + timestamp；或路径 A 的本地 proxy。`HttpReqFrame` 建议加 `ts` 字段以分离"隧道时延"和"LLM 处理时延"。
2. **决策层**：控制端一个跨 child 的事件循环（多 `Sandbox` 的 tunnel 事件集中器），实现 poll 语义；输出"对 child X 施加动作 Y"。
3. **执行层（最关键缺口）**：
   - sandbox API 加 `Pause/Resume`（cgroup v2 freezer，需先支持 cgroup v2）或 `Resize`（运行时改 cpu/mem limit）；
   - 恢复 `Checkpoint/Restore` RPC + runsc save/restore，以支持 memory 维度的释放；
   - 或提供显式 `Yield`（agent SDK 侧：告知控制端"我即将等 LLM，可调度"）。
4. **协议层**：延长或可配置 `HTTP_TIMEOUT`；考虑加流式帧（chunked transfer）以支持 SSE 不降级。

当前开源 v0.1.0 的真实状态：**观测层数据可得但无 API；执行层完全缺失**。所以"poll 调度"现在是**论证成立、系统未实现**。

---

## 8. 与你先前"stream 劫持"设想的关系

你之前（见 [stream 劫持调研](./20260812T160000Z-akernel-yuanrong-source-structure-stream-hijack-checkpoint-survey.md)）设想的是"劫持 sandbox 出向的 model API stream → checkpoint → resume"。本文揭示的 ReverseTunnel 是一条**更干净、且部分已实现**的替代路径：

| 维度 | stream 劫持设想 | ReverseTunnel 路径 |
|---|---|---|
| 取得可观测性 | 透明代理 + 根 CA 植入 + TLS 解密 | child 主动访问 loopback，TLS 在控制端终止（**无需劫持**） |
| 安全/合规 | 需在 sandbox 植私有 CA，多租户噩梦 | child 只看到 loopback HTTP，无证书注入 |
| 触发方式 | 观察到"model API 流量"才动作 | child 调用 loopback 即是显式信号（确定性，无歧义） |
| 当前实现 | 0% | 数据路径已实现（TunnelServer/TunnelClient/Traefik 全在） |
| 缺口 | 全栈缺 | 仅缺观测 API + 执行面 |

**结论：ReverseTunnel 是"让父亲观测 child 的 LLM 调用"的正确实现方式，远优于 stream 劫持。** 你最初听到的"stream 劫持"说法，在源码层面应当被修正为"经 ReverseTunnel 的显式服务注入 + 控制端终止 TLS"。论文写作时建议用 ReverseTunnel 这条路径作为论证基础。

---

## 9. 最终判断

1. **ReverseTunnel 语义**：把控制端（"父亲"）侧的 HTTP/HTTPS 服务暴露给 child sandbox；child 访问 loopback，请求经反向 WebSocket 汇入控制端，TLS 在控制端终止。方向是 child → 控制端，**不是劫持**。
2. **"父亲知晓 child 对 LLM 的调用"——成立**：数据必然流经控制端，完整明文可见，per-request 可区分。缺的是 request 级 hook API（数据有，接口无）。
3. **"LLM 调用即 I/O、poll 调度、child 等待期被调度走"——概念完全成立，且 CPU 维度已隐式达成（child 等待时天然不占 CPU）；memory 维度（缩资源/checkpoint）和决策/执行面在开源 v0.1.0 缺失。**
4. **相对 K8s/SigmaOS 的差异化论点成立**：把 LLM 调用变成控制端可观测的、带语义的 I/O 事件，覆盖 Pod 级（K8s 太粗）与单机 proc 级（SigmaOS 不面向 LLM）之间的空档。ReverseTunnel 是实现这个论点的天然底座。
5. **论文写作建议**：以 ReverseTunnel（而非"stream 劫持"）作为可观测 I/O 模型的实现基础；如实标注观测 API 与执行面（resize/checkpoint/fork）为 in-progress/roadmap；把 600s 超时和 SSE 降级作为 design tradeoff 公开讨论。

---

## 10. 源码证据索引

### 10.1 ReverseTunnel 方向与语义
- [HttpReverseTunnel docstring：Expose SDK-side service inside sandbox](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/types.py:158)
- [README：sandbox code call service reachable from SDK；TLS at SDK side](/mnt/u/hukeyang/AKernel/akernel/sdk/python/README.md:200)
- [示例：target=父亲本地 http.server](/mnt/u/hukeyang/AKernel/akernel/sdk/python/examples/reverse_tunnel.py:48)

### 10.2 SDK 控制端
- [start_reverse_tunnel：拼 wss URL + 启动 TunnelClient](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_openyuanrong.py:237)
- [TunnelClient 跑在父亲进程 daemon 线程](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/sandbox/tunnel_client.py:222)
- [_handle_http：转发 target，TLS 终止，完整明文在手](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/sandbox/tunnel_client.py:475)
- [httpx ssl context（TLS 在控制端）](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/sandbox/tunnel_client.py:273)
- [body 整段缓冲（非流式）](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/sandbox/tunnel_client.py:500)

### 10.3 sandbox 内 server（rrt-runtime / Python）
- [start_tunnel_server：sandbox 内起 TunnelServer](/mnt/u/hukeyang/AKernel/akernel/sdk/python/akernel_sdk/_instance.py:374)
- [Rust rrt-runtime 端口绑定 0.0.0.0:ws + 127.0.0.1:http](/mnt/u/hukeyang/AKernel/yuanrong/api/rust/rrt-daemon/src/runtime/tunnel.rs:201)
- [启动期绑定（feature gate RRT_TUNNEL_WS_PORT）](/mnt/u/hukeyang/AKernel/yuanrong/api/rust/rrt-daemon/src/runtime/mod.rs:890)
- [HTTP_TIMEOUT=600s、504 超时](/mnt/u/hukeyang/AKernel/yuanrong/api/rust/rrt-daemon/src/runtime/tunnel.rs:38)
- [per-request id + multi-plex + 并发信号量](/mnt/u/hukeyang/AKernel/yuanrong/api/python/yr/sandbox/tunnel_client.py:401)

### 10.4 frontend / Traefik 路径
- [prepareSandboxTunnel 注入 RRT_TUNNEL_WS/HTTP_PORT](/mnt/u/hukeyang/AKernel/yuanrong/frontend/pkg/frontend/api/sandbox/handler.go:828)
- [TunnelSpec/TunnelInfo（frontend 拥有 tunnel 细节）](/mnt/u/hukeyang/AKernel/yuanrong/frontend/pkg/frontend/api/sandbox/handler.go:190)
- [/direct /tunnel 路由（入向，非劫持）](/mnt/u/hukeyang/AKernel/yuanrong/frontend/pkg/frontend/api/sandbox_direct.go:55)
- [Traefik 规则 PathPrefix(/{safeID}/{port})](/mnt/u/hukeyang/AKernel/yuanrong/functionsystem/functionsystem/src/function_proxy/local_scheduler/traefik_registry/traefik_registry.cpp:99)

### 10.5 调度执行面缺口（开源 v0.1.0）
- [sandbox API 仅 6 RPC，无 Pause/Resume/Resize](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:22)
- [CgroupHandler 无 Freeze/Thaw](/mnt/u/hukeyang/AKernel/akernel/src/sandboxd/internal/cgroupops/cgroup.go:22)
- [cgroup v1 only（AGENTS.md）](/mnt/u/hukeyang/AKernel/akernel/AGENTS.md)
- [Checkpoint/Restore 被裁（见 fork 调研 §5）](/mnt/u/hukeyang/AKernel/AKernel_Docs/Surveys/20260812T163000Z-yuanrong-fork-launch-akernel-sandbox-path-survey.md)

### 10.6 论文定位
- [checkpoint-on-idle 规划（等 LLM 时释放资源）](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:403)
- [两级调度：Realm scheduler + node daemon](/mnt/u/hukeyang/AKernel/AKernel_Docs/Paper_outline.md:303)
- [Agent Process 状态机 Idle→Checkpointed](/mnt/u/hukeyang/AKernel/AKernel_Docs/AKernel_Paper_Structure.md:377)
- [spawn/fork/exec 语义待统一](/mnt/u/hukeyang/AKernel/AKernel_Docs/Introduction_outline.md:83)
