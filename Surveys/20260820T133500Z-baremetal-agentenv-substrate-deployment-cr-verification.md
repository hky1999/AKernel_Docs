# BareMetal 上 AgentEnv 与 Agent Substrate 部署手册及 C/R 性能验证

> 实验时间:2026-08-20(UTC)
> 文档时间戳:`20260820T133500Z`(v2,补充 microVM 路径与并发 C/R)
> 执行位置:BareMetal(`147.139.138.166`,SSH alias `BareMetal`,root)
> AgentENV commit:`7f4a9b9`(main,2026-08-20 拉取)
> Agent Substrate commit:`bbf9d440`(main,2026-08-20 拉取)
> 前置调研:[部署前置条件 survey](20260812T024137Z-agentenv-agent-substrate-deployment-prerequisites-survey.md)
> 部署路径:AgentEnv = 单节点 Docker(Firecracker data plane);Substrate = Kind + gVisor(gvisor-default SandboxConfig)

## 1. 结论摘要

两套系统在 BareMetal 上均部署成功,C/R(暂停/恢复)功能与状态保留均验证通过,并补充了 Substrate microVM(Cloud Hypervisor)路径与两侧并发 C/R。同机对比(均为 5 轮实测):

| 指标 | AgentEnv(Firecracker) | Substrate gVisor Actor | Substrate microVM Actor |
|---|---:|---:|---:|
| checkpoint(pause/suspend) | **62–143 ms**(中位 ~70 ms) | **242–290 ms**(含 gRPC 控制面往返) | **171–202 ms** |
| restore(resume/冷请求) | **80–110 ms** | **378–444 ms**(含请求 parking + ateapi 调度 + snapshot 拉取) | **496–519 ms** |
| restore 后热路径 | exec 正常 | 后续请求 ~6 ms | 后续请求 ~5–6 ms |
| 从 committed/golden snapshot 恢复 | 83 ms(`aenv start <snapshot>`) | ~397 ms(新 Actor 首请求) | ~503 ms(新 Actor 首请求) |
| 状态保留 | ✓(10 轮计数器文件单调递增) | ✓(file counter 14→22,内存计数保留) | ✓(file counter 4→12,内存计数保留) |

并发 C/R(8 路同时,详见第 6 节):

| 8 路并发 | AgentEnv pause / resume | Substrate gVisor 冷恢复 | Substrate microVM 冷恢复 |
|---|---:|---:|---:|
| 全部完成 wall time | 86–284 ms / 110–203 ms | 520 ms | 600 ms |
| 单操作 p50 | 79–188 ms / 100–173 ms | 453–510 ms | 551–592 ms |
| 吞吐 | 93–158 ops/s | ~15 restores/s | ~13 restores/s |
| 前提 | 无(Firecracker warm pool 自动扩容) | **需要 ≥N 个空闲 warm Worker**,否则溢出请求 park 满 5 s 后 503 | 同左 |

定性差异:AgentEnv 的 C/R 是**数据面原语**(直接操作 Firecracker snapshot,毫秒级,vm_state.bin 增量仅 20 KB);Substrate 的 C/R 是**控制面编排**(suspend → Full snapshot 落 RustFS → 请求触发 resume → Worker claim → gVisor/CH 恢复),单次延迟高一个量级,但换来 Actor 逻辑身份、请求 parking 和 location-transparent 路由。两者不在同一层,数字不可直接互换。microVM 的 suspend 反而快于 gVisor(175 vs 250 ms),restore 略慢(~510 vs 400 ms);microVM snapshot(zstd,含 base-id/state.json/rootfs-upper)共 29 MiB。

## 2. 宿主机基线与准备

### 2.1 BareMetal 初始状态

| 项目 | 值 |
|---|---|
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.8.0-136-generic(满足 AgentEnv 6.8+ 要求) |
| CPU / 内存 | 104 vCPU / 187 GiB |
| 磁盘 | `/dev/vda3` 295 GB(实验时余 258 GB) |
| `/dev/kvm` | 存在(root:kvm) |
| `ublk_drv` | 初始未加载,`modprobe` 后正常,`/dev/ublk-control` 出现 |
| 网络 | GitHub / dl.google.com 均直连可达 |

### 2.2 安装的依赖(全部在 BareMetal 上)

```bash
apt-get install -y docker.io          # docker 29.1.3, systemd enable --now docker
modprobe ublk_drv                     # 已在 /etc/modules-load.d 持久化(由 aenv docker-setup.sh 写入)
sysctl -w net.ipv4.ip_forward=1
# Go 1.24.6 官方 tarball → /usr/local/go
# kubectl v1.33.3 → /usr/local/bin/kubectl
# kind v0.27.0 → /usr/local/bin/kind
# ko v0.19.1 → go install github.com/google/ko@latest
# aenv CLI 0.1.3 → scripts/install-cli.sh
```

Docker:`storage=overlayfs cgroup=systemd`。

## 3. AgentEnv 部署手册(单节点 Docker 路径)

工作目录:`/root/AgentEnvWorkSpace`。

### 3.1 步骤

```bash
# 1. 克隆源码
git clone https://github.com/kvcache-ai/AgentENV.git /root/AgentEnvWorkSpace/AgentENV

# 2. 宿主机准备(加载 ublk_drv、持久化、sysctl tuning)
bash /root/AgentEnvWorkSpace/AgentENV/scripts/docker-setup.sh

# 3. 拉取 server 镜像
docker pull ghcr.io/kvcache-ai/aenv-server:latest

# 4. 启动(注意:占用宿主机 8000 端口)
docker run -d --name aenv-server --restart unless-stopped \
  --device /dev/kvm --privileged -v /dev:/dev \
  -v aenv-data:/workspace \
  -p 8000:8000 \
  ghcr.io/kvcache-ai/aenv-server:latest

# 5. 安装 CLI 并配置凭据
bash /root/AgentEnvWorkSpace/AgentENV/scripts/install-cli.sh
docker exec aenv-server cat /workspace/env/secrets/api-key   # 形如 e2b_...
mkdir -p /root/.config/aenv
cat > /root/.config/aenv/credentials <<EOF
url = "http://localhost:8000"
api_key = "<上一步的 key>"
EOF
chmod 600 /root/.config/aenv/credentials
```

说明:`aenv auth` 是交互式 prompt,非交互 SSH 下无法使用;CLI 实际读取的是 TOML 凭据文件 `~/.config/aenv/credentials`(见 `crates/aenv/src/auth.rs`),直接写入等价。`-v aenv-data:/workspace` 把数据根持久化为 docker volume(官方示例不带,去掉后容器重建即丢 API key 与模板)。

### 3.2 验证

```bash
curl http://127.0.0.1:8000/health          # HTTP 204
aenv ls                                     # []
time aenv pull ubuntu:22.04 --name ubuntu   # ~11 s(含 OverlayBD 转换)
time aenv start -d --timeout 3600 ubuntu    # warm 模板启动 74 ms
aenv exec <SID> uname -a                    # Linux instance 6.1.175(Firecracker guest kernel)
```

模板构建后 `01a01f43-0e73-…`,sandbox 默认 2 vCPU / 1024 MiB / 65536 MiB disk。启动后 Firecracker pool 会预热(低水位 2 个 warm VM)。

### 3.3 运行态事实

- aenv-server 容器内 `/workspace/env/snapshot-store` 与 `/workspace/env/persisted-sandboxes` 实测各 ~1.5 GiB(含模板层与 paused state)。
- pause 后产生的 `vm_state.bin` 仅 **20 KB**(增量内存快照;全量 guest RAM 不落盘)。
- 宿主机可直接 `pgrep firecracker` 观察容器内 Firecracker 进程(pause 时计数减 1)。

## 4. Agent Substrate 部署手册(Kind + gVisor 路径)

工作目录:`/root/AgentSubstrateWorkSpace`。

### 4.1 步骤

```bash
# 1. 克隆源码
git clone https://github.com/agent-substrate/substrate.git /root/AgentSubstrateWorkSpace/substrate
cd /root/AgentSubstrateWorkSpace/substrate

# 2. 创建 Kind 集群 + 本地 registry(localhost:5001)
bash hack/create-kind-cluster.sh

# 3. 部署 ate-system(CRD、atecontroller、ateapi、atelet、atenet、Valkey×6、RustFS、证书控制器)
bash hack/install-ate-kind.sh --deploy-ate-system    # ko 构建 + kustomize,全程 ~5 min

# 4. 部署 demo counter(WorkerPool + ActorTemplate + golden snapshot 构建)
bash hack/install-ate-kind.sh --deploy-demo-counter

# 5. 安装 kubectl 插件
go install ./cmd/kubectl-ate && ln -sf /root/go/bin/kubectl-ate /usr/local/bin/kubectl-ate

# 6. 创建 atespace 和 Actor
kubectl ate create atespace demo
kubectl ate create actor my-counter-1 -a demo --template=ate-demo-counter/counter
# Actor 初始即 ACTOR_STATE_SUSPENDED(仅控制面记录 + golden snapshot)

# 7. 暴露路由并发请求(注意端口,见 4.2)
kubectl -n ate-system port-forward svc/atenet-router 18000:80 &
curl -X POST -H 'Host: my-counter-1.demo.actors.resources.substrate.ate.dev' http://localhost:18000/
# → hello from: 169.254.17.2 | preserved memory count: N | preserved file counter: M
```

### 4.2 踩坑记录(重要)

1. **端口冲突**:AgentEnv 已占用宿主机 8000。`port-forward ... 8000:80` 会静默失败(bind: address already in use),而 `curl localhost:8000` 命中的是 **AgentEnv API,返回 401**,极易误诊为 Substrate 鉴权问题。任何对 atenet-router 的 port-forward 必须避开 8000(本实验用 18000)。
2. `kubectl ate` 默认使用 kubeconfig current-context;脚本导出 `KUBECTL_CONTEXT=kind-kind`,手工操作时显式 `--context kind-kind`。
3. Actor **不会**在短空闲后自动 suspend(counter 模板未配置 idle suspend;`timeoutSeconds: 30` 是请求超时)。显式 checkpoint 用 `kubectl ate suspend actor <name> -a demo`。
4. Kind 节点被脚本标记了 `ate.dev/sandboxClass=microvm`(因 Docker 内可打开 `/dev/kvm`),但 demo counter 的 WorkerPool 实际使用 `ateom-gvisor` 镜像 + `gvisor-default` SandboxConfig(gVisor 路径)。

### 4.3 运行态事实

- 组件:`ate-system` 14 个 Pod 全 Ready;WorkerPool `counter` 3/3;ActorTemplate `counter` Ready 并生成 goldenSnapshot(`onPause: Full`、`onCommit: Data`,location 指向 RustFS `ate-snapshots` bucket)。
- RustFS snapshot store 实测 1.6 MiB:golden + per-actor snapshot 均为 zstd 压缩的 `pages.img.zstd / checkpoint.img.zstd / pages_meta.img.zstd / manifest.json`。
- Valkey 6 实例 TLS cluster(3 master + 3 replica),RustFS 各 1 GiB PVC,均由脚本自动完成。

## 5. C/R 验证方法与结果

### 5.1 AgentEnv(pause/resume + committed snapshot)

方法:guest 内维护递增计数文件 `/tmp/cr-count`;每轮 exec 递增 → 计时 `aenv pause` → 确认 Firecracker 进程消失(pgrep 计数减 1)→ 计时 `aenv resume` → 回读计数校验。脚本:`/root/AgentEnvWorkSpace/cr-bench.sh`。

5 轮干净内存 + 5 轮(guest `/dev/shm` 写 512 MiB 脏页):

| 场景 | checkpoint (ms) | restore (ms) | 状态保留 | FC 进程消失 |
|---|---:|---:|---|---|
| round 1–5(基线) | 143 / 62 / 63 / 85 / 70 | 81 / 80 / 86 / 110 / 97 | 5/5 ✓ | 5/5 ✓ |
| round 1–5(+512 MiB 脏页) | 257 / 68 / 57 / 54 / 66 | 101 / 88 / 86 / 104 / 100 | 5/5 ✓ | 5/5 ✓ |

脏内存对延迟几乎无影响——与 `vm_state.bin` 仅 20 KB 一致,pause 采用增量内存快照,而非全量 RAM dump。

Committed snapshot 路径(可复用、可跨节点):

```text
aenv snapshot create <SID> --name ubuntu-cr-bench   6970 ms(内存 + OverlayBD 层提交到 snapshot-store)
aenv start -d ubuntu-cr-bench                          83 ms(新 sandbox 从 committed snapshot 恢复)
恢复后 cat /tmp/commit-marker.txt                      → "committed" ✓
```

### 5.2 Agent Substrate(suspend/resume Actor + golden snapshot)

方法:每轮显式 `kubectl ate suspend actor my-counter-1 -a demo`(计时 = checkpoint)→ 确认 `ACTOR_STATE_SUSPENDED` → 首个 HTTP POST 计时(= restore + 请求,经 atenet-router parking → ateapi resume → Worker claim → gVisor 恢复)→ 第二个请求为热路径。脚本:`/root/AgentSubstrateWorkSpace/cr-bench.sh`。

| round | suspend_ms | 冷请求_ms(restore) | 热请求_ms | suspend 后状态 | 冷请求响应 |
|---:|---:|---:|---:|---|---|
| 1 | 290 | 418 | 6 | SUSPENDED | mem=1, file=14 |
| 2 | 246 | 439 | 6 | SUSPENDED | mem=1, file=16 |
| 3 | 242 | 426 | 6 | SUSPENDED | mem=1, file=18 |
| 4 | 253 | 378 | 6 | SUSPENDED | mem=1, file=20 |
| 5 | 251 | 444 | 7 | SUSPENDED | mem=1, file=22 |

file counter 跨轮单调 +2(每轮冷+热各一次请求),内存计数恢复为 suspend 前值 → 快照状态保留 ✓。

新 Actor(golden snapshot 恢复路径):

```text
kubectl ate create actor my-counter-2 ...        96 ms(纯控制面记录)
首个请求(golden snapshot restore)              397 ms,响应 file=1(全新状态)✓
```

### 5.3 对比解读

1. **AgentEnv 的 restore(~100 ms)与 Substrate 的冷请求(~400 ms)差约 4 倍**,但后者包含:atenet-router 请求 parking、ateapi Actor resume 决策、Worker slot claim、从 RustFS 拉取 zstd snapshot、gVisor 恢复、再回程转发。它衡量的是"suspend 的 Actor 被一个请求唤醒到首个字节"的完整控制面+数据面链路;AgentEnv 的 resume 只是单节点数据面原语。
2. **Substrate suspend(~250 ms)比 AgentEnv pause(~70 ms)慢**,同样是 Full snapshot:前者要走 ateapi→atelet→ateom 控制链并把 zstd 对象写入 RustFS;后者是 node-local 增量落盘。
3. **committed/golden 恢复两者相当**(83 ms vs ~397 ms,仍是控制面开销差异):AgentEnv 从本地 posix snapshot-store 恢复,Substrate 从对象存储恢复并保留 Actor 逻辑身份。
4. AgentEnv 的 snapshot create 6.97 s 是唯一秒级操作(commit 全量 artifact),但它一次性换取此后 83 ms 量级的可重复恢复——与 Substrate 的 golden snapshot 模式同构。
5. 两个系统的 C/R 语义都验证了状态正确性(文件+内存),且 pause/suspend 后原执行体消失(AgentEnv: Firecracker 进程终止;Substrate: Worker slot 归还池),资源占用归零的 claim 均成立。

### 5.4 Substrate microVM 路径(Cloud Hypervisor + Kata assets)

部署(在 4.1 的 ate-system 基础上):

```bash
apt-get install -y unzip    # assemble.sh 需要,首次踩坑
cd /root/AgentSubstrateWorkSpace/substrate
bash hack/run-microvm-demo-kind.sh
# 该脚本:幂等重跑 ate-system → 下载装配 assets(kata-static 4.0.0 约 1.9 GB、
# cloud-hypervisor、virtiofsd、kata-kernel/image/config)→ stage 到 RustFS
# → 部署 counter-microvm WorkerPool(ateom-microvm 镜像,默认 2 replicas)
kubectl --context kind-kind wait --for=condition=Ready \
  actortemplate/counter-microvm -n ate-demo-counter-microvm --timeout=900s
kubectl ate create actor mv-counter-1 -a demo \
  --template=ate-demo-counter-microvm/counter-microvm
```

验证 microVM 真实性(易误判:atelet 把所有 staged assets 都命名为 `runsc-<sha>` 前缀,包括 cloud-hypervisor 本体):

```text
宿主机 pgrep 可见(与 Actor uid 对应):
  virtiofsd --socket-path=/run/vc/vm/<actor-uid>/virtiofsd.sock
            --shared-dir=/run/kata-containers/shared/sandboxes/<actor-uid>/shared
  cloud-hypervisor --api-socket /run/vc/vm/<actor-uid>/clh-api-restore.sock
WorkerPod 镜像 = ateom-microvm;SandboxConfig class=microvm(assets digest 校验通过)
```

C/R 结果(Actor `mv-counter-1`,2 vCPU / 512 MiB,5 轮,方法同 5.2):

| round | suspend_ms | 冷请求_ms(restore) | 热请求_ms | 状态保留 |
|---:|---:|---:|---:|---|
| 1–5 | 175 / 202 / 177 / 171 / 175 | 519 / 512 / 496 / 518 / 504 | 5–6 | file counter 4→12 ✓ |

golden snapshot 恢复(新 Actor `mv-counter-*` 首请求)503 ms。microVM checkpoint 日志显示实际分阶段耗时(pause ~1.8 ms + durable_dir 打包 ~1.8 ms + teardown ~30 ms,数据量小时开销都在控制链)。RustFS 中 microVM snapshot 共 29 MiB(gvisor counter 仅 ~0.2 MiB/actor)。

## 6. 并发 C/R 基准

### 6.1 AgentEnv:8/16/32 sandbox 同时 pause / resume

方法:N 个 sandbox(ubuntu 模板,2 vCPU/1 GiB)全部 running 后,N 个 `aenv pause` 并发发出,记 wall time 与每操作延迟;resume 同理。脚本:`/root/AgentEnvWorkSpace/conc-scale.sh`。

| N | pause wall / p50 / max(ms) | resume wall / p50 / max(ms) | pause 吞吐 | resume 吞吐 |
|---:|---|---|---:|---:|
| 8 | 86 / 79 / 83 | 110 / 100 / 107 | 93/s | 73/s |
| 16 | 125 / 96 / 120 | 117 / 106 / 113 | 128/s | 137/s |
| 32 | 284 / 188 / 268 | 203 / 173 / 194 | 113/s | 158/s |

要点:

- 32 路并发 pause 的 wall time(284 ms)仅约为单次延迟的 4 倍,Fan-out 基本线性;吞吐稳定在 93–158 ops/s。
- 并发 burst 后 AgentEnv 的 Firecracker warm pool 自动扩容(测试后留有 64 个 warm FC stub,每个 RSS 仅 ~2 MiB,总开销 0.1 GiB)。
- 32 轮并发后回读状态文件 32/32 保留(8 路三轮均 8/8)。

### 6.2 Substrate:8 Actor 同时 suspend / 冷恢复

方法:WorkerPool 先扩到 8(`kubectl scale workerpool ... --replicas=8`,Deployment 随之 rollout),8 个 Actor 并发 `kubectl ate suspend`,待全部 SUSPENDED 后 8 个冷请求并发发出。脚本:`/root/AgentSubstrateWorkSpace/conc-cr-bench.sh`(gVisor)与 `conc-cr-bench-mv.sh`(microVM)。

**干净条件(≥8 个空闲 Worker,残留 Actor 已全部 suspend)**:

| | gVisor | microVM |
|---|---:|---:|
| 并发 suspend wall(三轮) | 349–465 ms | 110–250 ms |
| 并发冷恢复 wall | 520 ms | 600 ms |
| 冷恢复单操作 p50 | 453–510 ms(6/8 成功;2 个失败见下) | 551–592 ms(8/8 成功) |

gVisor 轮中 2 个异常均为测试夹具问题而非容量问题:`conc-4` 处于 `ACTOR_STATE_RESUMING` 过渡态被拒 403(单独重试 200/398 ms),`conc-5` 是早前删除的残留 Actor(404)。

**饱和条件(空闲 Worker < 并发数)——关键运维发现**:最初的并发轮次中,每轮 8 个冷请求有 1 个稳定耗时 ~5.01 s 并以失败告终:

```text
503 actor demo/mvc-8 unavailable: no free workers available  (parked 5.005 s)
```

机制:atenet-router 的请求 parking 预算 `DefaultParkedRequestBudget = 5 s`(100 ms × 1.1ⁿ 退避,`ingress/parking.go:29`);WorkerPool replicas 固定、**不会按 resume 需求自动扩容**,溢出请求 park 满 5 s 后 503。生产使用必须按预期并发恢复速率配置 warm Worker 数,或实现 pool 自动扩缩容。AgentEnv 侧无此限制(Firecracker warm pool 随负载自动扩)。

另:并发 suspend 时 ateapi `ResumeActor` RPC 本身 ~460–490 ms(内含等待 worker 恢复),控制面重负载下的 5 s 尾延迟来自 parking 层而非 ateapi。

## 7. 清理与复现

```bash
# AgentEnv(保留部署则跳过)
docker rm -f aenv-server && docker volume rm aenv-data

# Substrate
cd /root/AgentSubstrateWorkSpace/substrate && kind delete cluster --name kind

# 复现 C/R 基准
bash /root/AgentEnvWorkSpace/cr-bench.sh <sandbox-id> 5
bash /root/AgentEnvWorkSpace/conc-scale.sh <N> pause "<sid1> <sid2> ..."   # 并发
kubectl -n ate-system port-forward svc/atenet-router 18000:80 &
bash /root/AgentSubstrateWorkSpace/cr-bench.sh my-counter-1.demo.actors.resources.substrate.ate.dev 5
bash /root/AgentSubstrateWorkSpace/conc-cr-bench.sh 8 demo 3               # gVisor 并发
bash /root/AgentSubstrateWorkSpace/conc-cr-bench-mv.sh 8 demo 3            # microVM 并发
```

## 8. 遗留与后续

- ~~Substrate microVM 路径未测~~ → 已测(5.4 节);~~并发 C/R 未测~~ → 已测(第 6 节)。
- **跨节点恢复确认为当前不可测**(与预期一致):Substrate 部署在单节点 Kind(控制面本身支持 snapshot 从对象存储异地恢复,验证需多 worker 节点);AgentEnv 默认 snapshot repository 是节点本地 `posix_fs`,多节点需共享 POSIX 存储或 OSS/S3 backend,当前单机部署不具备条件。
- 未测:fork 语义(AgentEnv)、更大内存 footprint(>512 MiB 脏页)下两类 snapshot 的伸缩、Substrate WorkerPool 自动扩缩容(当前 replicas 固定,饱和时溢出请求 503,见 6.2)。
- 并发测试中观察到 Actor 状态机偶发卡在 `ACTOR_STATE_RESUMING`(conc-7 长时间未迁移,期间不可 suspend/delete);早期版本并发操作下需注意状态过渡窗口。
- AgentEnv API 无鉴权(BareMetal 单机可接受,勿暴露公网);Substrate kind 部署的 RustFS/Valkey 凭据写在开发 manifest 中,仅限本地。
- 本实验测量的是单节点、无竞争(除注明饱和场景)、warm cache 条件下的延迟;磁盘为云盘 `/dev/vda3` 而非 NVMe,绝对值偏保守。
