# gVisor PR#14228(runsc 可写层外部化 C/R)优化效果评估 —— 与 AgentEnv / Agent Substrate 同机实测对比

> 评估时间:2026-08-21(Asia/Shanghai);文档时间戳 `20260821T041720Z`
> 评估对象:[PR#14228](https://github.com/google/gvisor/pull/14228) `runsc: keep writable-layer filestores out of the checkpoint image`
> 参考底稿:`AKernel_scheduler/docs/CR/pr-description.md`、`AKernel_scheduler/docs/CR/issue.md`(2026-08-19/21)
> 实测环境:BareMetal(104 vCPU / 187 GiB / Ubuntu 24.04 / kernel 6.8,cgroup v2,数据盘为云盘非 NVMe),三套系统同机同日测量
> 二进制:PR 分支 `bin/runsc`(release-20260810.0-109-g52b4de8a9159-dirty)vs 上游 main `bin/runsc-main`(release-20260810.0-95-g5d1328306b1b-dirty,即 PR 文档所述对比基线)
> 复现脚本:`/root/cr-bench-bm/scripts/pr_eval.sh`(本次新写,日志 `preval/logs/run.out`)、`/root/AgentEnvWorkSpace/aenv_eval.sh`、`aenv_fork_eval.sh`、`/root/AgentSubstrateWorkSpace/cr-bench.sh`(本次修复硬编码 bug)
> 前序:[OverlayBD vs AKernel survey](20260820T163100Z-agentenv-overlaybd-spawn-cr-cow-vs-akernel-survey.md)、[BareMetal 部署与 C/R 验证](20260820T133500Z-baremetal-agentenv-substrate-deployment-cr-verification.md)

## 1. 结论摘要

**这个 PR 的优化是渐近(asymptotic)级别的,不是常数因子优化**:它把 runsc C/R 对可写层 S 的依赖从 O(S) 内容序列化改为 O(1) 生命周期绑定(FICLONE),restore 侧从 O(S) 彻底消失,park 侧只剩元数据 + FICLONE。同机今日实测(冷 restore,md5 全绿):

| S(可写层) | main park | **PR park(全程/冻结窗口)** | main restore | **PR restore** | main image/工件 | PR 工件 |
|---:|---:|---:|---:|---:|---:|---:|
| 64M | 48–53 ms | 336–459 ms / 52 ms | 276–289 ms | 300–302 ms | 130 MB | 66 MB |
| 1G | 399–452 ms | 5.6–6.1 s / 57 ms | **5.6–5.7 s** | **297–311 ms** | 2050 MB | 1026 MB |
| 4G | 1.7–2.1 s | 2.0 s / — | **22.9 s** | **342 ms** | 8194 MB | 4098 MB |

- **restore:4G 时 67×(22.9s → 342ms),且对 S 完全平坦**(1G 与 4G 只差 45ms)。这是 PR 的核心收益,直接决定它能否作为 park/unpark 调度的 quantum。
- **park 的诚实口径要拆开说**:parkd 全程含 P3 宿主 `sync` 写回(1G ≈ 5.7s,云盘带宽决定,O(dirty)),但**冻结窗口(P4 ckpt+带内 FICLONE)只有 52–57ms 且对 S 平坦**——写回可异步剥离(数据已在宿主 page cache,崩溃风险窗口与 main 的 image delalloc 相当)。main 的 park 计时不含 image fsync(PR 文档实测过 ENOSPC 边缘静默截断),两边口径都"不含持久化保证"时:PR ~150–310ms vs main 0.4–2.1s。
- **fanout(S=1G,N=8):逐实例 102–113ms(FICLONE 0–1ms + restore ~105ms),XFS df 增量 ≈ 0**——8 份 1GiB"虚拟副本"免费。
- 两个诚实的边界:(a) S=64M 时 main park 反而更快(48ms vs parkd 编排总 336ms;PR 的固定开销在 python 编排器,冻结窗口本身 52ms 与 main 相当);(b) PR 冷 restore 后**首次触碰冷数据仍付 O(S) 磁盘读**(1G md5 首触 5.3s vs main 2.3s,后者 restore 时已顺带把页读热)——PR 把 O(S) 从"恢复关键路径"挪到"首次访问",对恢复后只碰热区的真实负载是净赢,对恢复后立刻全量扫盘的负载不赢。

**三系统同日同机对比(S=1GiB 脏口径)**:

| | **PR#14228**(runsc 数据面) | **AgentEnv**(FC+OverlayBD) | **Substrate**(控制面,小状态) |
|---|---|---|---|
| checkpoint | 冻结窗口 **57ms**(全程含写回 5.7s) | pause **824–1305ms**(O(dirty) 实锤) | suspend 227–350ms(gVisor)/ 176–185ms(microVM) |
| restore | **297–311ms**(对 S 平坦) | resume 91–765ms | 冷请求 383–429ms / 501–520ms |
| 派生 N 份 | **103ms/份,磁盘 +0** | fork API 772ms,**磁盘 +0.9×S(一份共享 mem 层,一次性)** | 无 fork;新 Actor 走 golden ~400–500ms/个 |
| 对 S 的扩展性 | **O(1)(唯一平坦者)** | O(dirty) 增量,首代/fork 全量 | 未测大 S;snapshot 为 zstd 序列化内容,预期 O(S) |

定位差异:PR 是**数据面原语**(runsc flag + 编排器),AgentEnv 是**自带数据面的沙箱服务**,Substrate 是**K8s 原生 Actor 控制面**(其下仍是 gVisor/CH 的内容序列化 C/R)。PR 恰好补的是 Substrate 这类系统底层最缺的那块:Substrate 的 suspend/cold 数百 ms 里控制面开销占大头,但其 snapshot 机制(zstd 全量序列化)在 GB 级可写层上会撞上与上游 runsc 完全相同的 O(S) 墙——PR 落地上游后,Substrate 式控制面 + PR 式数据面是自然组合。

## 2. PR 机制一句话回顾

可写层本来就是一个宿主文件(匿名稀疏 filestore,gofer fd 持活)。PR 让 C/R 只绑定它的**生命周期**而非序列化其**内容**:

- `checkpoint --skip-filestore-pages`:私有 MemoryFile 只存段元数据(ContentExternal),state image 退化为纯内存态模板(~1MB,只读可复用);
- `--filestore-snapshot-dir`:冻结窗口内由 sentry 自己 FICLONE 快照 + `filestores.json` 清单(尺寸+采样指纹);
- `restore --filestore-adopt-dir [--filestore-clone-on-adopt]`:领养工件(指纹校验),默认再 FICLONE 一份私有 CoW → 一次 (image, 快照) 对可恢复出 N 个写隔离沙箱;
- 正确性防线:non-adopted/undersized 大声失败、fresh-start 拒绝、teardown 不再 punch hole 外部化工件、seccomp 只放行 FICLONE ioctl。

细节见 `AKernel_scheduler/docs/CR/pr-description.md`(已提交 google/gvisor#14228,8 commits,rebase 到 master `80336ad54`)。

## 3. PR vs 上游 main:今日同机实测

方法:alpine + `--overlay2=all:dir=XFS`,systrap,`--compression=none`;负载 = 可写层内 S MB urandom(`dd conv=fsync`);每格 2 轮;restore 前 `drop_caches`(冷口径);双向 md5 硬校验。PR 侧 park/unpark 走 parkd(P1 定位 fd → P2 holder → P3 sync → P4 带内 ckpt+FICLONE → P6 teardown → P7 清单)。

### 3.1 主表

| BUILD | S | park_ms(全程) | restore_ms(冷) | 工件 MB† | md5 首触 ms | md5 |
|---|---|---:|---:|---:|---:|---|
| main | 64M | 53 / 48 | 289 / 276 | 130 | 165–166 | ok |
| **pr** | 64M | 459 / 389 | 300 / 302 | 66 | 187–209 | ok |
| main | 1G | 399 / 452 | **5693 / 5602** | 2050 | 2252–2291 | ok |
| **pr** | 1G | 6063 / 5580 | **311 / 297** | 1026 | 5304–5449 | ok |
| main | 4G | 2117 / 1691 | **22892 / 22878** | 8194 | 9004–9178 | ok |
| **pr** | 4G | 2035 / — | **342** | 4098 | 22522 | ok |

† main = checkpoint image(含可写层页);pr = state image + FICLONE 工件(parkd 目录 du;其中 state image 本身 ~1MB,见 PR 文档,其余是快照工件——reflink 与源共享 extent,du 计为实占)。

### 3.2 parkd 分相(为什么 park 全程比冻结窗口慢)

1GiB 轮次 parkd 日志(park-pr-1024-1.out):

```text
P2 holder:            50ms
P3 writeback(sync): 5753ms   ← 宿主写回 1G 脏页,云盘带宽决定,O(dirty)
P4 ckpt+带内FICLONE:   57ms   ← 真正的冻结窗口,对 S 平坦
P6 teardown:          37ms
P7 manifest:           OK
```

- **冻结窗口(P4)= 52–57ms**,与 PR 文档的 63–68ms 一致;这是调度意义上的 park 代价(沙箱停摆时长)。
- P3 是**持久化选择**而非机制代价:main 的 park 计时同样不含 image fsync(其 ENOSPC 静默截断正是因为 delalloc 未刷)。两边都按"不含刷盘保证"口径:PR 关键路径 ≈ 150–310ms(编排器 python 开销为主,sandboxd Go 化后可降),main = 0.4–2.1s(O(S) 序列化 CPU)。
- 若要求崩溃一致,P3 写回不可省——但它是**异步可剥离**的(后台 flush,不阻塞冻结),且与 main 的 O(S) 序列化不同:写回是顺序大 IO,序列化是 CPU + 逐页 fault(配额下退化 3–9×,PR 文档实测 512M 配额 restore 8.9–9.6s vs PR 0.24–0.29s)。

### 3.3 md5 首触:PR 的 O(S) 没有消失,只是搬了位置

| S | main md5 首触 | PR md5 首触 |
|---|---:|---:|
| 64M | 165 ms | 187–209 ms |
| 1G | 2.3 s | **5.3 s** |
| 4G | 9.0–9.2 s | **22.5 s** |

PR 冷 restore 后可写层页不在 page cache(adopt 的 FICLONE 工件从未被读过),首触按云盘冷读付全价;main 的 restore 过程把整层写进新 filestore,页是热的。**含义**:park/unpark(恢复后跑热区)PR 全赢;恢复后立即全量扫描/哈希全盘的负载,总耗时(restore+首触)两边同为 O(S),PR 不占优——这是"懒恢复 vs 预恢复"的普遍权衡,不是缺陷,但评估时要按真实负载的热区比来算。

### 3.4 fanout(常数成本派生)

S=1G park 一次 → `parkd fanout --n 8`:

| 实例 | FICLONE | restore | 总 | md5 |
|---|---:|---:|---:|---|
| #1–#8 | 0–1 ms | 102–113 ms | 102–113 ms | 8/8 ok |

**XFS df 增量 = -20 KiB(即 0)**:8×1GiB 虚拟副本全部 extent 共享。与 PR 文档 N=64(174–206ms/实例,磁盘 0)一致;本机今天负载略高(Kind/AgentEnv 同机运行)。

### 3.5 效果评估结论

1. **restore 渐近消除**:67×@4G,平坦;这是 PR 的不可替代价值,直接把 C/R 变成可用的调度 quantum。
2. **park 冻结窗口 O(1)**(57ms),全程含写回则 O(dirty) 但异步可剥离且是纯顺序 IO。
3. **image O(S)→~1MB**,附带消灭 ENOSPC 静默截断故障模式。
4. **新增 fork 语义**(常数成本、零磁盘),上游此前完全没有。
5. **小 S 无优势甚至略亏**(64M:park 全程 336–459 vs main 48ms;restore 300 vs 276ms)——PR 的编排固定开销(~150–300ms,python parkd)在小 S 时主导。目标场景是 GB 级长寿命沙箱,与 PR 动机一致。
6. 主要依赖:XFS/btrfs reflink(ext4 降级 fd-hold、无 fork)、编排器配合(parkd/sandboxd)、flags 实验性默认关闭。社区定位风险:上游是否接受"生命周期绑定"语义(见 issue.md §5 的五问)。

## 4. AgentEnv 同口径补测(1GiB 脏)

方法:ubuntu 模板 sandbox,guest 内 `dd 1GiB urandom conv=fsync`,pause/resume ×3,fork 走 HTTP API(`POST /sandboxes/{id}/fork`,X-API-Key,`{"count":N}`)。脚本 `/root/AgentEnvWorkSpace/aenv_eval.sh` / `aenv_fork_eval.sh`。

### 4.1 pause/resume(1GiB 脏)

| round | pause_ms | resume_ms | md5 |
|---|---:|---:|---|
| 1–3 | 827 / 1305 / 824 | 91 / 765 / 115 | 3/3 ok |

对照干净内存时的 62–143ms:**1GiB 脏使 pause 退化到 ~0.8–1.3s(≈1.2ms/MB)**,与 OverlayBD 增量内存层机制(KVM dirty tracking + `process_vm_readv` 压层)一致——AgentEnv 的 pause 是 O(dirty),每轮把全部脏页物化进 mem 层(每轮 pause 留下一个 20KB vm_state.bin + 一层 mem overlaybd commit)。resume 91–765ms 波动大(首轮冷读 mem 层)。

### 4.2 fork(1GiB 脏,count=2/3)

| 项 | 值 |
|---|---|
| fork API 返回 | 772 ms(3 个 child) |
| 磁盘实占增量(du) | **937 MiB ≈ 0.9×S(fork×3);fork×2 树确认单份共享** |
| 工件 | `managed-snapshots/<src>/<snap>/mem_overlaybd/overlaybd.commit` 916–933 MiB(**一份,全部 child 共享**)+ `rootfs/snapshot.commit` 1024 MiB(父 upper rename 封存,非新增字节) |
| 语义 | child 内 1GiB 文件可见 ✓ |

**修正前一份 survey 的结论**:此前(100MiB 实验)记录"同一父的多个 fork 各自物化一份 mem 层"——本次 1GiB 两次实验(fork×3 du 增量 = 单份 915MiB;fork×2 工件树仅一个 mem_overlaybd.commit)证实 **mem 层是同父所有 child 共享一份**,每个 child 只新建私有空 upper。即 AgentEnv fork 磁盘成本 = O(dirty) **一次性**(而非每 child 一份),时延 ~0.8s@1G。仍是线性物化(真字节落盘、写带宽付全价),与 PR 的 FICLONE(元数据级、0 字节)有本质差距,但好于此前的判断。

## 5. Substrate 复测

gVisor Actor(my-counter-1)与 microVM Actor(mv-counter-1,Cloud Hypervisor)各 suspend→冷请求循环:

| 路径 | suspend_ms | 冷请求_ms | 热请求_ms | 状态保留 |
|---|---|---:|---:|---|
| gVisor ×5 | 350 / 235 / 227 / 250 / 286 | 412 / 383 / 429 / 395 / 406 | 6–12 | file counter 单调 ✓ |
| microVM ×4(修正脚本后) | 180 / 180 / 185 / 176 | 519 / 520 / 501 / 509 | 5–6 | ✓ |

与 2026-08-20 数据一致(gVisor 242–290/378–444;microVM 171–202/496–519)。

**顺带发现并修复了 bench 脚本 bug**:`/root/AgentSubstrateWorkSpace/cr-bench.sh` 第 9 行把 suspend 硬编码为 `my-counter-1`,只有 Host header 用了 `$1`——对 microVM actor 跑该脚本会挂起 gVisor actor、而目标 actor 一直热着(冷请求变 6ms 热请求,内存计数连续增长、日志无 CheckpointWorkload RPC)。已改为按 `$1` 推导 ACTOR。昨日部署文档 §5.4/§6.2 的 microVM 数据与修正后重跑一致,结论不受影响。

另:microVM suspend 日志分相(pause 1.9ms + durable_dir 打包 1.7ms + teardown 42ms,checkpoint RPC 全程 71ms)——小状态下 Substrate 的 suspend/cold 耗时几乎全是控制面链(ateapi→atelet→ateom、RustFS 读写、Worker claim),不是数据面。

## 6. 三系统对比分析

### 6.1 同一问题的三种姿态

| | PR#14228 | AgentEnv | Substrate |
|---|---|---|---|
| C/R 语义 | 状态模板 + 工件领养(生命周期绑定) | 增量层链(内容物化,逐脏页) | 全量 zstd 序列化(内容搬运) |
| 隔离边界 | gVisor sentry | Firecracker microVM | gVisor / Cloud Hypervisor |
| checkpoint 成本模型 | O(1) 冻结 + 可剥离 O(dirty) 写回 | O(dirty) 物化,层链增量 | O(S) 序列化 + 对象存储 |
| restore 成本模型 | **O(1) + 首触 O(冷区)** | O(mem 层读) | O(S) 反序列化 + 控制面 |
| 大 S 扩展性(实测) | 4G 平坦 | 1G 已 0.8–1.3s(线性) | 未测(机制上 O(S)) |
| memcg 配额下 | 平坦(工作集在宿主页缓存) | 未测(FC 进程 anon,预期受压) | 未测 |
| 派生(fork) | 常数 103ms/份,磁盘 0 | 一次性 O(dirty) 0.9×S,~0.8s@1G | 无(golden 新建 ~0.5s/个,每份全量) |
| 并发 | 64 park 零失败/restore 风暴 686ms(doc) | 32 并发 pause 284ms(113–158 ops/s) | 8 路冷恢复 520–600ms,**饱和即 503**(WorkerPool 固定) |
| 生态成本 | reflink FS + 编排器 + 实验性 flag | 自带全套,但 privileged + ublk + 单点 daemon | K8s 全家桶(Valkey/RustFS/证书) |

### 6.2 数字对照(S=1GiB 脏,同日同机)

| 操作 | PR#14228 | AgentEnv | Substrate gVisor | Substrate microVM |
|---|---:|---:|---:|---:|
| 挂起(冻结/suspend) | **57ms**(窗口) | 824–1305ms | 227–350ms | 176–185ms |
| 恢复(到可服务) | **297–311ms** | 91–765ms | 383–429ms | 501–520ms |
| 恢复后首触全量 1G | 5.3s(冷读) | ~热(mem 层刚读) | n/a(状态小) | n/a |
| 派生 8 份(1G) | **~0.8–0.9s 总,磁盘 0** | ~0.8s + 0.9G 磁盘(一份共享) | n/a | n/a |

读法:

1. **小脏集稳态 C/R,AgentEnv 最快**(pause 62–143ms / resume 80–110ms,增量只付脏页);PR 的 unpark 固定 ~300ms(sentry 重建 + 领养),不吃脏集但也不因小脏集变快。
2. **S 增大时唯一不退化的是 PR**。AgentEnv 1G 脏 pause 已 1s 量级(线性);Substrate 的 snapshot 是 zstd 全量序列化,counter 级状态无感,GB 级可写层会撞上与上游 runsc 相同的墙(其 microVM 路径底层正是 Kata/CH + 序列化)。PR 文档的 memcg 数据(512M 配额 restore 8.9–9.6s vs PR 0.24–0.29s)进一步说明 overcommit 高压调度只有 PR 形态可用。
3. **fanout/派生是 PR 的独有语义优势**:8×1G 零磁盘、103ms/份,AgentEnv fork 共享一份 mem 层(修正后)但仍是线性物化真字节,Substrate 无对应物。
4. **Substrate 的价值不在数据面速度**而在 Actor 身份、请求 parking、位置透明路由——它的 suspend/cold 数百 ms 中数据面只占 ~70ms(日志分相)。PR 若落上游,Substrate 式控制面把底层 runsc C/R 换成 PR 流程,即可继承 O(1) restore 与 fork(其 WorkerPool 固定 replicas 的 503 饱和问题仍在,需配自动扩缩)。
5. **跨节点**:PR 把 image(~1MB)与可写层快照解耦为两个独立搬运对象(reflink 共享存储或差分传输);AgentEnv committed snapshot 走 posix/OSS + P2P(6.97s 全量 commit 一次,之后 83ms 恢复);Substrate 天然对象存储化(golden ~400ms)。三者中 PR 的跨节点搬运量最小(image 1MB + 差分)。

### 6.3 对 PR 本身的评估意见(供 PR/讨论参考)

- 收益真实且量级正确:restore 渐近消除 + 冻结窗口 O(1) + image ~1MB + fork 语义,全部本次同机复现(md5 全绿)。
- 建议在 PR 性能段落强调两个口径,避免 reviewer 误读:(a) park 全程 vs 冻结窗口(写回可剥离);(b) 恢复后冷数据首触仍是 O(S) IO——"O(1) restore"指的是到进程可运行,不是到全部数据常驻内存。
- 小 S 场景(≤64M)PR 无优势,编排固定开销主导;这符合其 GB 级目标定位,但值得在 RFC 中明说。
- parkd(python)编排开销 150–300ms 在生产化(sandboxd Go)后应显著下降;当前数字对 PR 机制本身是保守估计。

## 7. 局限与未测

- memcg 配额矩阵、systrap/kvm/ptrace 平台矩阵、64 并发、3 代血统、soak/故障注入:本次未复测,沿用 PR 文档 + `bm_native.sh`/`bm_matrix.sh`/`bm_conc.sh` 既有结果(同机 2026-08-20,72 格全绿)。
- PR state image 单独尺寸未在本轮拆分(脚本输出的是 image+工件合计;~1MB 见 PR 文档)。
- AgentEnv >1GiB 脏、Substrate 大可写层(GB 级)未测——Substrate counter 模板无大状态写入路径,需自定义 workload 才能测,留作后续。
- 三系统负载形态不同(runsc=urandom 块文件;AgentEnv=ext4-in-overlaybd 写入;Substrate=应用级小状态),"S=1G 脏"对齐的是数据量而非写入模式;绝对值只做同数量级对照。
- 本次 PR 二进制为 rebase 前 build(52b4de8a9);PR 已 rebase 到 master `80336ad54`(tip `1985ab80b`),变更仅为 proto 元数据格式适配,机制与性能口径不变(rebase 后 pgalloc_test + bm_native 6/6 复验过)。

## 8. 证据索引

- 头对头 + fanout 原始输出:BareMetal `/root/cr-bench-bm/preval/logs/run.out`、`park-pr-*.out`、`fanout.out`;脚本 `scripts/pr_eval.sh`(2026-08-21 新增)
- 既有 R0 数据(72 格矩阵/并发/fork/故障注入):`/root/cr-bench-bm/{matrix,conc,fork,soak}/logs/`,入口 `scripts/bm_*.sh`;汇总 `AKernel_scheduler/docs/CR/issue.md` §1.3/§1.5
- AgentEnv:`/root/AgentEnvWorkSpace/aenv_eval.sh`、`aenv_fork_eval.sh` 输出(2026-08-21 12:11–12:14 +08);fork API `POST /sandboxes/{id}/fork`
- Substrate:`/root/AgentSubstrateWorkSpace/cr-bench.sh`(本次修复 ACTOR 推导);ateom checkpoint 分相日志 `kubectl -n ate-demo-counter-microvm logs deploy/counter-microvm`
- PR/issue 底稿:`AKernel_scheduler/docs/CR/pr-description.md`、`issue.md`;上游 PR <https://github.com/google/gvisor/pull/14228>
