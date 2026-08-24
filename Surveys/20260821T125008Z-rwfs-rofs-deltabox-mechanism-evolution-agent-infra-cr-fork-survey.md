# RWFS(PR#14228)× ROFS(nydus/distill-fs)× DeltaBox:机制源码对照与演进方向 Survey

> 调研时间:2026-08-21(Asia/Shanghai)
> 文档时间戳:`20260821T125008Z`
> 性质:源码级机制对照 + 演进方向分析。引用三方源码/论文:
> - **RWFS**:自研 gVisor fork PR#14228(`scheduler-adopt-filestore`,`~/AKernelWorkspace/gvisor`,8 个 C/R commit;机制与实测详见 [PR 评估 survey](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md));
> - **ROFS**:`~/AKernelWorkspace/AKernel/src/distill-fs`(2026-08-21 全量源码分析)+ nydus 上游事实(取自 [0806 survey](20260806T104027Z-akernel-nydus-distillfs-gvisor-erofs-dragonfly-faasnet-stateful-agent-survey.md) 的 web 核实部分);
> - **DeltaBox**:Dong 等,SJTU IPADS + 华为,arXiv:2605.22781v2(`References/Dong 等 - 2026 - DeltaBox *.pdf`;对照分析见 scheduler 仓库 1100Z 可行性文档)。
> 上游设计文档(层图/DS 形态/remote fork/live migration):scheduler 仓库 0900Z / 1040Z 可行性分析。

---

## 1. 结论摘要

三方在源码层面是**同一个模式的三种实现**——「不可变模板 + CoW/按需实例化 + 校验过的身份」——但写方向、粒度、代际语义和分发面完全不同:

| 维度 | DeltaFS/L1(DeltaBox) | RWFS(PR#14228) | ROFS(distill-fs/nydus) |
|---|---|---|---|
| 模板形态 | guest 内 XFS reflink 层栈(overlayfs 模块管理) | host XFS 上 filestore sparse 文件的 FICLONE 快照 + `filestores.json` | registry blob(RAFS v5 chunk 内容寻址)+ bootstrap |
| 实例化 | rename 降级保 reflink 边 → ioctl 层栈切换(0.07ms) | FICLONE CoW(p95 ~1ms)+ restore(163–311ms) | FUSE 按需缺页(mount warm ~20ms) |
| CoW 粒度 | 4KB extent(reflink) | 4KB extent(reflink) | chunk 级(RAFS ~1MiB;distill 缓存栈 4MiB) |
| 写路径 | guest overlay upper | 实例私有 CoW fault | **无**(FUSE 层无 write handler) |
| 代际语义 | `checkpoint_gen` + **快照树**,任意点 O(1) 回滚 | 链式(park/resume),fork=clone-on-adopt | 无版本(镜像不可变) |
| 分发 | 无(单机) | 无(本地优先;跨节点=整份传输) | **有**(registry + ChunkDB + P2P) |
| C/R 感知 | 是(CRIU 配合) | 是(runsc 原生) | **零感知**(无任何 checkpoint 钩子) |

一句话:**RWFS 有"快"没"传",ROFS 有"传"没"写/代际",DeltaBox 有"代际/回滚"但被钉死在单机+定制 guest 内核。** 演进方向就是把三者的长处拼进同一套层图:RW 模板 chunk 化后**复用 distill-fs 已有的 ChunkDB/serve-chunk/P2P 设施**做跨节点分发;环境准备层(L1)物化为 RO 层(nydus blob/EROFS)获得集群级分发;ROFS 侧补 v6/BlobDevice 合并读与 C/R 感知预热;调度侧补 DeltaBox 式的代际树、hot-park、轻量路由与工作负载感知 GC。

---

## 2. 三方机制源码级解剖

### 2.1 DeltaBox L1+L2:reflink 层栈 + rename 降级(对照基准)

- guest 内 XFS reflink:相邻 checkpoint 间写放大 O(4KB),跨 N 代**传递性块共享**(一个未改 extent 永远只占一个物理块)。
- 关键机制是 **DeltaFS(改过的 overlayfs 内核模块)**:checkpoint 时把 upper **rename 降级**为 RO lower——rename 保 extent map、不重物化,reflink 边因此跨代保留;`checkpoint_gen` 计数 + ioctl 运行时热切换层栈(0.07ms),回滚 = 切回历史层栈组合,O(1)。
- 打开文件的 lazy switch、async-warm(`/proc/pid/mem` 预触 CoW 页)、CRIU 异步 dump(tmpfs + SIGSTOP ~11ms)、冻结模板池 fork restore(1.86ms)。
- 代价(1100Z §4):必须定制 guest 内核模块 + agent 适配(NPD FIFO、quiescent 自 fork);纯单机;不支持网络 I/O 回滚。

### 2.2 RWFS:PR#14228 在 gVisor fork 里的三块代码

| 机制 | 位置 | 行为 |
|---|---|---|
| checkpoint 外置 | `pkg/sentry/pgalloc/save_restore.go:122-143,274-320` | `--skip-filestore-pages` 使可写层私有 MemoryFile 只存 `ContentExternal` 段元数据 → image ~1MB,磁盘线 O(1) |
| 冻结窗快照 | `pkg/sentry/kernel/filestore_snapshot.go:44-86` | 冻结窗口内对每个宿主 filestore `ioctl(FICLONE)` + 写 `filestores.json`(resource ID/size/chunk 数/首尾 4K 采样 sha256);实测窗口 52–57ms 对 S 平坦 |
| restore 领养 | `runsc/container/filestore_snapshot.go:158-266` | `--filestore-adopt-dir [--filestore-clone-on-adopt]`:按 ResourceID 领养 + exact size + 采样指纹双校验(loud-fail),clone-on-adopt 默认再 FICLONE 私有副本 → 一模板 N 写隔离沙箱(103–290ms/份,磁盘 +0) |

性质(与 DeltaBox 的本质差异):

1. **快照是"文件级 reflink 拷贝"而非"rename 降级"**——每次 park 产生一个新快照文件,与源(及其前代快照)通过 reflink 链共享 extent。XFS 上 FICLONE 的传递性同样成立,但**代际关系只存在于编排层(parkd 的目录命名),runsc 本身无 `checkpoint_gen`、无层栈概念、无任意点回滚语义**。
2. **模板与实例都是 host 普通文件**——这是与 DeltaBox(guest 内模块)相反的边界选择,换来:stock host 内核、任意 guest 镜像、sentry 可写层本就是 host 文件所以能外部化。
3. **跨节点 = 整份搬运**(M5a/M5d 已证位置无关,但 tar/rsync 口径 O(S));manifest 的 sampled sha256 是指纹雏形,不是 chunk 索引。

### 2.3 ROFS:distill-fs 源码分析(2026-08-21,全文见附录引用)

单二进制 `distill_fs`,四个子命令(`mount`/`serve-chunk`/`gc-chunk`/`stats-chunk`,src/cli.rs:161-167)。要点:

**读路径(miss 链)**:

```
FUSE read → (raw) IndexDB 区间查询 → ChunkDB 命中即拷贝
        → miss:Unix socket 问本地 chunk server(serve-chunk)
        → peer PrefetchChunkBlocking(P2P,Redis 归属索引,≤3 owner,校验后落库)
        → 仍 miss:cache 稀疏文件(4MiB bitmap 边车)→ backend(registry/OSS/S3/localfs/http-proxy)
        → 异步入队去重(有界队列,满则丢弃)
(nydus 模式:chunk 边界/压缩/摘要全部来自 RAFS 元数据,src/image/nydus.rs:344-375,665;
  ChunkDB 存解压后 chunk;miss 从 blob cache 或 backend 取压缩区间再解压,nydus.rs:514-574)
```

**共享与去重**:全局内容寻址 ChunkDB(LMDB,map 100GiB,src/backend/chunkdb.rs:43,391-393),多 mount 进程共用同一 `--chunk-db-dir` 即共享;`IndexDB` 按镜像存 offset→checksum 区间(indexdb.rs:105-129);chunk 持久化后对 cache 文件 `FALLOC_FL_PUNCH_HOLE` 回收(cache.rs:498-519)。跨节点 = chunk 级 P2P(明文自定义帧协议,peer.rs:51-73,2819-2874)。

**关键边界(对演进最重要)**:

1. **纯只读**:FUSE 只实现 lookup/getattr/read/readdir/xattr,无 write/create/mkdir(raw.rs:210-334;nydus.rs:577-799)——没有任何"把上层写回模板"的通道;
2. **对 C/R 零感知**:grep `checkpoint|restore|snapshot|gvisor|filestore` 无命中;缓存对进程重启持久(bitmap + 稀疏文件 truncate(false) 重开,LMDB 天然持久,cache.rs:315-367)——"restore 后缓存热"只是隐式副作用,不是设计;
3. **RAFS v5-only**:v6/加密 chunk/batch chunk 直接 `Unsupported`(nydus.rs:189-195,332-343)——v6 的 EROFS-compatible 元数据和内核页缓存路径用不上(0806 survey §3.2);
4. **逐 chunk FUSE 读,无 BlobIoVec 合并/预取**:标准 nydusd 能合并连续 chunk range + prefetch,distill_fs 没走 `BlobDevice` 流程;且 blob cache 以 4MiB extent 缓存,稀疏小读有放大(0806 survey §4);
5. **无预取策略**:"prefetch" 仅指 miss 时向 peer 拉取,没有任何预测性预热。

### 2.4 nydus 上游(对照参照系)

- RAFS v5:chunk info 含 digest/blob index/压缩偏移,典型 chunk ~1MiB,用户态解析;
- RAFS v6:compact inode 与 **EROFS 兼容**,允许内核 EROFS 接手 pathname/inode/page-cache,nydusd(或等价 daemon)主要按 fscache 请求回填 data blob——即「内核页缓存共享 + 用户态只做数据面回填」;
- 标准 nydusd 有 FUSE / virtio-fs / fscache 多前端,BlobCache 可围绕 `BlobChunkInfo` 合并连续 range 并 prefetch。

这三条正是 distill-fs 当前没吃到的东西:v6 内核路径、chunk 合并读、预取。

---

## 3. 统一层图与缺口分析

沿用 0900Z §3.2 的层图,把三方机制摆进去:

```
L0  任务镜像        nydus RO(distill-fs FUSE,v5)          ← 分发✓(registry+ChunkDB+P2P)
L1  环境准备层      filestore FICLONE 快照(PR#14228 模板)   ← 实例化✓(CoW) 分发✗(整份)
L2+ 运行时增量     实例私有 CoW + statefile 内存线 O(RSS)   ← 链式 park/resume
旁挂 agent 镜像    nydus RO(多 harness 复用同一 L1)        ← 分发✓
代际/回滚          ✗(无 checkpoint_gen、无快照树)           ← DeltaBox 有,我们缺
```

从源码看,缺口可以精确到"哪块代码不存在":

| 缺口 | 源码证据 | DeltaBox 对应物 |
|---|---|---|
| L1 模板跨节点 chunk 分发 | PR manifest 只有 sampled 指纹;parkd/YR 链路是 zip 整份(0900Z §1.1) | 单机无此问题;但 distill-fs 的 ChunkDB/P2P **是现成积木** |
| 写层→RO 层发布(seal) | distill-fs 无 write handler;runsc 无"快照→nydus blob/EROFS"转换(0806 survey §4 gap) | rename 降级(但降级后仍是 reflink 层,不是可分发格式) |
| 代际语义/任意点回滚 | runsc 无 generation 概念;快照树只存在于编排层命名 | `checkpoint_gen` + 层栈 ioctl + 快照索引树 |
| 零写入跳过 | 每次 park 必 FICLONE(57ms 常数虽小,仍有 sync 写回 O(dirty) 在总账里) | 轻量检查点路由(62% 事件免 dump) |
| restore 后预热 | 无(冷首触 1G md5 5.3s 实测) | async-warm |
| 冻结不拆毁档 | park = 拆 sandboxd 实例;resume 必付 163–311ms sentry 重建 | 冻结模板池(1.86ms fork) |
| 生命周期 | TTL 1800s 硬编码 + sweeper | reachability-aware GC |

---

## 4. 演进方向(核心)

### D1. RW 模板 chunk 化:把 filestore 快照接进 distill-fs 的 ChunkDB/P2P ⭐ 最高杠杆

**洞察:跨节点分发层不用新建——distill-fs 已经有全套。** ChunkDB(内容寻址、LMDB、节点级共享)+ serve-chunk(TCP/Unix socket)+ PeerClient(Redis 归属索引、RTT 打分、checksum 复验)就是 0900Z 形态 C 缺的那个"DS chunk 接口 + 重组器"的现成实现(至少在节点/集群粒度上,DS 侧可作为统一的 object directory 后端)。

设计:

```
park(现状不变):      FICLONE 快照落本节点 XFS + filestores.json
上传(lazy,新):       对快照做稀疏感知分块(非零 extent 按 4MiB 或对齐 chunk),
                     chunk → ChunkDB(peer 网络自动可分发);manifest 升级为
                     全量 chunk 索引(offset→checksum),替代 sampled 指纹
跨节点 restore(新):  本地无快照 → 查 chunk 索引 → 缺失 chunk 经 P2P/DS 拉取
                     → 重组稀疏文件(保洞!禁止再物化)→ FICLONE 实例化
```

正确性要点(1100Z 洞察 1 的直接应用):

1. **重组必须保 extent 边**:稀疏重组(只写非零块),否则跨代传递性 reflink 共享在重组文件上丢失,目标节点磁盘变 O(S);
2. **adopt 前校验不降级**:exact size + 全量 chunk checksum(比 PR 的采样指纹更强,且 checksum 本来就要算——进 ChunkDB 必须);
3. chunk 粒度选择:distill 缓存栈固定 4MiB(`backend/mod.rs:29`),与 XFS extent(粗粒度,CoW 时常 64KB–1MB 连续)匹配度尚可,但 env 演进(依赖增量)场景下可评估对齐 FICLONE 后实际脏 extent 边界的变长分块——先 4MiB 起步,索引里记 extent 布局。

收益:remote fork 的模板获取从 O(S) 整份变为 O(未命中 chunk);同源模板演化(env 升级)跨节点只传脏 chunk;fanout 目标节点互为 peer(≤3 owner 上限需调)。

### D2. seal 路径:L1 模板物化为 RO 层(环境准备的"一次性付清,永久复用")

场景一(装完依赖 checkpoint)的模板会被**成百上千次 fork、跨节点复用**——对这种"高频复用低频修改"资产,FICLONE 快照(每节点各一份、节点内 reflink)不如 RO chunk 格式(集群一份、按需拉)。演进:

```
env-prep park → FICLONE 快照(节点内模板,即时可用,fork 走 clone-on-adopt)
            → (seal,异步)快照 → 去零/压缩 → nydus blob + bootstrap(或 EROFS image)
            → registry/DS 发布 → 任何节点 distill-fs/EROFS 按需挂载
两个消费路径并存:
  本地热路径:restore --filestore-adopt-dir(写隔离 CoW,0.2s/份)
  集群冷路径:RO 层挂载 + 新起沙箱(或:RO 挂载为 L1' + 空 L2)
```

这是 0806 survey 已识别的 gap("当前没有 seal upper → 发布 Nydus delta 的代码")的正式化。工程上相当于写一个"host 文件 → RAFS blob"的打包器(nydus-image 已有 `--source-dir` 打包能力,输入换成快照文件/重组目录即可),压缩去零后传输量通常 << S。取舍:seal 是异步的一次性成本,不进 park 关键路径;模板不可变性由 RO 格式天然保证(比 read-only open 的约定更强)。

### D3. ROFS 自身演进:v6 / BlobIoVec 合并读 / C/R 感知预热

1. **RAFS v6 支持**(读路径迁移到 nydus `BlobDevice` 流程,README 已声明这是前置):收益 = chunk 合并读(BlobIoVec)+ 为将来内核侧(fscache/EROFS,或 gVisor sentry 内 EROFS 直解 v6 元数据)留路。AKernel 已有 sentry 内 EROFS 实现(`pkg/erofs`),v6 的 EROFS-compatible 布局意味着**sentry 理论上可直接吃 v6 bootstrap**——这是把 distill-fs 从数据路径上摘掉(元数据进 sentry、数据面只剩 chunk 回填 daemon)的长期方向;
2. **预取(async-warm 的 ROFS 对偶)**:restore 完成后的首个 LLM 等待窗内,后台按 blob 顺序预触热 chunk(标准 nydusd BlobCache prefetch 语义,distill-fs 补上);同时攻冷首触尾巴(RW 侧预读 filestore 热 extent + RO 侧预读 chunk,同一个"async-warm 窗口调度器"可统一做);
3. **C/R 感知钩子**(最小集):mount 时接受"预热清单"(bootstrap 的 prefetch table 已有格式可借);ChunkDB 命中统计暴露给调度器,作为暖节点放置信号(1040Z §5.1 的 chunk 命中率输入,数据源就在 stats-chunk)。

### D4. 代际语义:链 → 树,以及零写入跳过

近期(不动 runsc):

1. **零写入跳过**:park 前检查 filestore mtime/generation(实例自上次快照后零写 → 跳过 FICLONE,只存 state image;DeltaBox 轻量路由的 62% 命中率可参考,LLM 等待窗内实例大多零写);
2. **快照树在编排层**:parkd 目录已经按 checkpoint-id 组织,加 `parentCheckpointID`(0900Z §7.1 已设计)即成树;回滚任意点 = restore(历史 ckpt) + 从该点重新派生——语义上等价 DeltaBox 的快照索引树,只是回滚成本是 restore 常数(163–311ms)而非 0.07ms 层栈切换。

远期(若真要承接 MCTS 型高频回滚,1100Z §6 已列为可选):在 runsc 内引入 `checkpoint_gen` 与多层栈(filestore 快照链内嵌化)。当前用户三场景(分钟级粒度)不需要它,不建议近期投入。

### D5. hot-park 档位(攻 restore 常数)

signal 18 冻结但**不拆 sandboxd 实例**:短窗口 resume 走本地快路径(等价 DeltaBox 冻结模板池的"内存档"),水位/超时再降级为真 park。与 D1–D4 正交,纯 sandboxd/编排层工程(1100Z §3.2)。

### D6. 生命周期:工作负载感知 GC + 模板一等公民

- TTL 1800s 硬编码 → DS 分级 + 显式引用计数;保留策略感知 rollout 存活期(父 ckpt 保到子 rollout 结束)与 verifier 重试窗口(0900Z §8.3、1100Z 洞察 5);
- SnapshotInfo 扩展(0900Z §7.1):`artifactKind`/`manifestRef`(全量 chunk 索引)/`template+refcount`/`parentCheckpointID`——D1/D2/D4 的身份基础设施。

### 4.x 演进项与场景/文档的对应

| 演进项 | 攻的缺口 | 主要实现位置 | 依赖 |
|---|---|---|---|
| D1 chunk 分发 | 跨节点 O(S) | parkd/YR + distill-fs serve-chunk 复用 | PR 合入 + manifest 升级 |
| D2 seal→RO | 模板集群分发 | 新打包器(nydus-image 输入改造) | D1 的索引格式 |
| D3 ROFS 强化 | 冷首触/读放大/v6 | distill-fs BlobDevice 迁移 | nydus crates 已在依赖内 |
| D4 零写跳过+树 | park 常数/回滚语义 | 编排层(parkd/YR) | 无 |
| D5 hot-park | restore 常数 | sandboxd | 无 |
| D6 GC/身份 | TTL 硬编码 | YR SnapshotInfo | D1 |

---

## 5. 三个 RL/SFT checkpoint 场景的机制映射(源码视角)

### 5.1 场景一:SWE 环境准备 checkpoint × 多 agent harness

> 装完依赖 checkpoint(不再装依赖);agent 以自洽 OCI 镜像挂载;一个测试环境接多个 harness。

层图原生用法,机制映射到具体代码路径:

```
env-prep:   沙箱内装依赖 → park(FICLONE 快照 = L1 模板,filestore_snapshot.go)
本地 fork:  restore --filestore-adopt-dir --filestore-clone-on-adopt ×N
            (filestore_snapshot.go:158-266;103–290ms/份,extent 共享,模板只读打开不被污染)
跨节点:     D1(chunk 分发)或 D2(seal 为 RO 层)
agent 挂载: agent-i OCI → nydus RO(distill-fs mount,warm ~20ms)旁挂 —— rootfs 多层挂载
            与 snapshot 组合是 YR create 请求要补的语义(0900Z §6.1)
```

成本账:首次环境准备付一次全价;此后每个(环境 × agent)组合 = fork 常数 + agent 层挂载常数,磁盘零增量。harness matrix 组合爆炸的成本只有 env 模板一份——这是"一个测试环境接多个 harness"的经济基础。

### 5.2 场景二:agent 进行中 / LLM 失败现场续跑

> 过程中 checkpoint;LLM 连接失败从失败现场继续,不丢轨迹。

- **滚动 checkpoint**(P1.1 已实证):形态 C/D1 之后磁盘线降为常数(FICLONE 57ms + chunk 增量),只剩内存线 O(RSS)(~1.2ms/MB save)决定滚动间隔下限;
- **失败现场恢复**:transducer hold-forward + signal 24 定向唤醒已闭环(run3 PASS);restore 不继承 TCP → agent 重试 + transducer retry-serve 缓存交付;
- **轨迹不丢的双层保证**:records.jsonl 在 transducer 侧流式落盘(沙箱外),沙箱 C/R 只管 in-flight 进程/环境状态——两层合起来才是完整语义(0900Z §6.2);
- D4 零写入跳过直接利好此场景:LLM 等待窗内的滚动 ckpt 大多可免 FICLONE;D5 hot-park 让短故障(秒级)resume 免付 sentry 重建。

### 5.3 场景三:pre-verifier checkpoint

> 轨迹与执行结果在手,等验证拿 reward;verifier 出错可从 checkpoint 重跑。

- agent 完成 → park(32G 配额即时归还)→ verifier 独立沙箱/配额运行 → 失败则 restore(pre-verifier ckpt)重跑 verifier,agent 现场不重演;
- 峰值配额从「agent+verifier 相加」变「取 max」(pyright 类 verifier 自身 40G,t8-retry-1 OOM 的根因即此);
- D6 的 reachability-aware GC 在此最关键:pre-verifier ckpt 必须保到 verifier 成功为止,TTL 1800s 硬编码对慢 verifier 是正确性风险(不只是性能风险);
- 与 DeltaBox value-time test isolation 同构(独立验证),但我们粒度是全沙箱、零 agent 适配。

### 5.4 场景一 × DeltaBox 启示的额外注意

DeltaBox 的 rename-降级审计教训落到我们 fork:**快照链上任何"重写文件"的操作(压缩、resize、cp)都会断 reflink 边**。D1 的重组器与 D2 的 seal 打包器都必须只在"新用途"(分发副本)上物化,本节点 XFS 上的快照链原件永远保持 FICLONE 血统——这是写演进代码时要立的不变量。

---

## 6. Remote fork 与 live migration(引用 1040Z,源码视角补充)

### 6.1 Remote fork = 模板分发 + 本地实例化

三块积木已验证(park 产物位置无关 M5a/M5d、clone-on-adopt PR、KV 链 W3-C),缺的就是 D1:

- **暖节点**(模板 chunk 已驻):每实例仅 103–290ms restore+clone,与本地 fork 完全同价——"模板作为集群资产"的核心卖点;
- **冷节点**:首付 O(未命中 chunk)(D1)或整份 ~0.5s@10Gbps;
- 放置信号:D3-3 的 ChunkDB 命中统计 + DS chunk 视图 → 调度器成本感知选节点;CPUID 门在 restore 前 loud-fail;
- 预热调度:预知 fanout 计划时提前推 chunk(bandwidth-available 窗口),fork 时零等待。

### 6.2 Live migration = 静止窗口迁移(对 LLM 负载等价 live)

诚实定位:systrap 无脏页追踪 → 无 pre-copy;无 uffd → 无 post-copy。但 **transducer hold 窗口 = 天然静止期**(秒~分钟级 LLM 等待),停机 <1.5s 被完全掩盖。协议(源工件保留至目标健康确认 + replay 幂等防双活)见 1040Z §4.2–4.4,全部是编排层,无新数据面。D1/D5 落地后预算进一步压:暖目标 <1s、hot-park 窗口内甚至可探索"冻结驻留迁移"(不拆实例的 park 直接搬运)。

---

## 7. 路线图(合并 0900Z S0–S5 与本文 D1–D6)

| 阶段 | 内容 | 对应演进项 | 工作量 |
|---|---|---|---|
| S0(进行中) | 形态 A 收尾(TTL 可配、sweeper 解耦) | D6 前置 | 周内 |
| S1 | PR#14228 合入跟踪 + sandboxd 接线(manifest 入 SnapshotInfo) | 基础 | 周级 |
| S2 | **零写跳过 + hot-park**(纯编排/sandboxd,不动 runsc) | D4/D5 | 周级 |
| S3 | image 线入 DS(去 flate)+ manifest 升级全量 chunk 索引 | D6/D1 前置 | 周级 |
| S4 | **RW 模板 chunk 化**:分块器 + 重组器(保 extent 边)+ 接 serve-chunk/P2P;单模板 remote fork 端到端 | D1 | 月级 |
| S5 | seal→RO 打包器(nydus-image 输入改造)+ 场景一"H1 模板多 harness"YR create 语义 | D2 | 月级 |
| S6 | ROFS:BlobDevice/v6 迁移 + BlobIoVec 合并读 + async-warm 窗口调度器 | D3 | 月级 |
| S7 | 调度面整合(成本感知放置、模板感知准入、pre-verifier 串行化、reachability GC) | D6 | 月级 |

风险与开放问题:

1. **chunk 重组正确性**是 D1 的最大风险(稀疏保洞、extent 边、指纹全量化成本)——对冲:先做"整份传输 + 全量指纹索引"过渡版,chunk 拉取分阶段灰度;
2. **PR#14228 合入不确定性**(lifecycle-binding 语义上游讨论中)——fork 分支 72 格矩阵已自证,极端退回形态 A/B(image 线不受影响);
3. **内存线 O(RSS) 不因任何 FS 演进消失**(0900Z 底线重申):滚动 ckpt 间隔、迁移窗口下限都由它决定;
4. **distill-fs peer 协议明文、≤3 owner、无鉴权**(peer.rs:51-73,68)——进生产前要么加固要么换 DS 通道;
5. DeltaBox 的"任意点 O(1) 回滚"在 runsc 边界上不可得(restore 常数结构性),不要为对齐论文指标过度投入 D4 远期项。

---

## 8. 证据索引

### distill-fs(相对 `AKernel/src/distill-fs/`)

- 架构/子命令:src/cli.rs:161-167、src/main.rs:15-17、src/fs.rs:64-87
- 4MiB chunk 常量:src/backend/mod.rs:29
- CheckSum/磁盘格式:src/backend/chunkdb.rs:156-246
- ChunkDB/IndexDB LMDB:src/backend/chunkdb.rs:43,391-393;src/backend/indexdb.rs:105-129,175-217
- raw miss 链/P2P:src/backend/dedup.rs:458-550,660-687;src/backend/peer.rs:2819-2874(明文协议 peer.rs:51-73;owner 上限 peer.rs:68)
- 稀疏缓存 + bitmap + punch-hole:src/backend/cache.rs:207-210,348-368,498-519
- serve-chunk 双监听:src/cli.rs:490-563;Redis TTL 12h:src/cli.rs:42
- nydus 读路径/chunk 来自 RAFS 元数据:src/image/nydus.rs:344-375,514-574,665;digest:nydus.rs:304-308
- v5-only/加密/batch 拒绝:src/image/nydus.rs:189-195,332-343
- 无 write handler:src/image/raw.rs:210-334;src/image/nydus.rs:577-799
- GC(24h + LRU 90%→80%):src/backend/chunkdb.rs:53-55,1054-1090
- 缓存重启持久:src/backend/cache.rs:315-367

### gVisor fork(PR#14228)

- ContentExternal:`pkg/sentry/pgalloc/save_restore.go:122-143,274-320`
- 冻结窗 FICLONE + manifest:`pkg/sentry/kernel/filestore_snapshot.go:44-86`
- adopt/clone-on-adopt:`runsc/container/filestore_snapshot.go:158-266`
- filestore 创建(匿名 sparse):`runsc/container/container.go:1257-1283`
- sentry EROFS(0806):`pkg/erofs/erofs.go:222-231`;overlay 合成:`runsc/boot/vfs.go:570-760`

### nydus 上游(转引自 0806 survey,其时已 web 核实)

- RAFS v5/v6 对照:0806 survey §3.2(v6 EROFS-compatible、fscache 前端)
- BlobCache/BlobIoVec 合并读差异:0806 survey §4
- v6 layout 源码:<https://github.com/dragonflyoss/image-service/blob/master/rafs/src/metadata/layout/v6.rs>
- fscache service:<https://github.com/dragonflyoss/image-service/tree/master/service/src>

### DeltaBox / 设计文档

- DeltaBox 论文:arXiv:2605.22781v2(本地 `References/Dong 等 - 2026 - DeltaBox *.pdf`)
- 对照分析:scheduler `docs/feasibility/20260821T1100Z-DeltaBox论文对照与启示-可行性分析.md`
- 层图/DS 形态/场景:scheduler `docs/feasibility/20260821T0900Z-DS文件系统语义化与CR透明管理-可行性分析.md`
- remote fork/live migration:scheduler `docs/feasibility/20260821T1040Z-remote-fork与live-migration-可行性分析.md`
- PR 评估实测:[20260821T041720Z PR 评估 survey](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md)
