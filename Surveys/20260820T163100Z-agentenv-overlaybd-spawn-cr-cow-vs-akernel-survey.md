# AgentEnv(OverlayBD)spawn / C/R / CoW 语义与性能 vs AKernel C/R 实现 Survey

> 调研时间:2026-08-20/21(Asia/Shanghai)
> 文档时间戳:`20260820T163100Z`
> AgentENV commit:`7f4a9b9`(BareMetal `/root/AgentEnvWorkSpace/AgentENV`)
> AKernel:开源 v0.1.0(`/home/keyang/AKernelWorkspace/AKernel`)+ 自研 gVisor fork(`~/AKernelWorkspace/gvisor`,HEAD `4cdf0eb50`)+ sandboxd-pr16 + parkd/AKernel_scheduler + cr-bench 实测数据
> 实测环境:BareMetal(104 vCPU / 187 GiB / Ubuntu 24.04 / kernel 6.8,云盘非 NVMe);AgentENV 单节点 Docker 模式
> 前序:[BareMetal 部署与 C/R 验证](20260820T133500Z-baremetal-agentenv-substrate-deployment-cr-verification.md)、[AgentENV checkpoint 机制 survey](20260810T143748Z-agentenv-checkpoint-restore-golden-template-agent-substrate-akernel-survey.md)

## 1. 结论摘要

两套系统对"把一个可运行实例的状态变成可复用 artifact、再快速派生新实例"这个问题给出了两种不同解:

| 维度 | AgentEnv(Firecracker + OverlayBD/ublk) | AKernel(gVisor fork + filestore FICLONE 领养) |
|---|---|---|
| C/R 对象 | microVM(FC Diff 快照 + 增量内存层) | sentry 用户态内核 statefile + 宿主 filestore 文件 |
| spawn(同模板) | 31–66 ms/实例,16 并发 wall 89 ms | (开源 v0.1.0 无池化;README 40ms 为 planned) |
| checkpoint(park) | pause 62–143 ms(增量,~O(dirty)) | 分支 park 133 ms @S=1G(**对 S 平坦,O(1)**);stock 1236 ms |
| restore(unpark) | resume 80–110 ms | 分支 unpark 244 ms @1G(平坦);stock 2195 ms |
| fork/fanout | ~195–250 ms/个,一次性物化 **O(脏内存)** 共享 mem 层(2026-08-21 修正:同父 child 共享一份,非每 child 一份) | N×FICLONE + restore,**磁盘增量 0**,N=64 逐实例 ~195–290 ms |
| 写路径 CoW | 块级 512B segment(LSMT log-structured upper) | 文件级 reflink(XFS FICLONE,4K extent) |
| 内存共享 | rootfs 只读层共享;restore 时同 mem image 共享一个只读 ublk 设备 | rootfs mmap 共享 page cache;派生副本共享物理 extent |
| 内存恢复 | 全量映射 File backend + dirty tracking(uffd lazy 路径已写未启用) | statefile 反序列化 O(M)(1GiB ~470ms) |

一句话:**AgentEnv 把"快"做在增量(C/R 只付脏页代价),AKernel 把"快"做在零拷贝(FICLONE 使 park/unpark/fanout 对可写层大小完全不敏感)**。前者增量链有 compaction/继承链复杂度,后者要求 reflink 文件系统(XFS)且内存仍是 O(M) 序列化。

## 2. AgentEnv 数据面:spawn / C/R / CoW 机制

### 2.1 存储栈总览

- 单进程 `uvm-ublk-daemon` 集中管理所有 ublk 设备(`storage/ublk-daemon/src/main.rs:22-47`),Rust 版 OverlayBD(LSMT 格式)在用户态打开只读 lower 栈 + 可写 upper,经 `/dev/ublkbN` 挂给 Firecracker。
- **rootfs 与内存快照统一为 OverlayBD 分层块设备**——这是 AgentEnv 最核心的设计:内存也是层(`mem_overlaybd/overlaybd.commit`)。
- committed template 形态(`src/snapshot/types/artifacts.rs:15-26`、`posixfs/layout.rs:40-64`):

```text
repository/
  catalog/{records,aliases}                          # 记录与别名
  snapshots/{id}/{snapshot.json, firecracker-manifest.json, vm_state.bin}
  managed-layers/{digest}.overlaybd.commit           # 内容寻址共享层(与镜像层 digest 一致)
```

- OCI → OverlayBD:overlaybd-native 镜像零下载(记录 `RemoteLayer`,registryfs_v2 读时按 range 拉取);standard OCI 逐层 `overlaybd-create/apply/commit` 三步转换,产物按 digest 内容寻址缓存复用(`src/image/oci_image.rs:198-265,1297-1323`)。

### 2.2 spawn:块级 CoW 挂载

fresh 启动(`src/sandbox/firecracker/sandbox.rs:1192`)→ daemon `materialize_overlaybd_runtime`(`storage/ublk-daemon/src/runtime.rs:124-255`):模板只读层直接引用共享(managed-layers 硬链接/本地缓存),每沙箱新建磁盘文件 `upper.data + upper.index`(log-structured,默认 `UpperMode::LogStructured`,`src/sandbox/ublk/overlaybd.rs:76-82`)。

**CoW 粒度 = 512B 扇区级 LSMT segment**(`storage/overlaybd/src/lsmt/index.rs:8-16`、`file/types.rs:11`),非 4K 页。

### 2.3 pause/resume:增量内存层 + restack 根文件系统

pause 链(`sandbox.rs:647→681 snapshot_to_dir`):

1. FC `PATCH /vm Paused`;
2. `PUT /snapshot/create` 用 **`SnapshotType::Diff` 且 state-only**(vm_state.bin 因此只有 ~20 KB,`src/sandbox/firecracker/instance.rs:457-467`);
3. `GET /vm/dirty-memory-ranges`(KVM dirty tracking)→ **`process_vm_readv` 直接读 FC 进程内存**,只把 dirty ranges(4K 对齐)压成一层 overlaybd commit(并发 32,`src/sandbox/firecracker/overlaybd_snapshot.rs:42,759-783`);
4. rootfs 做 **restack**:把当前 log-structured upper 封存 rename 成 `rootfs/snapshot.commit`(同文件系统零拷贝),重开空 upper(`storage/overlaybd/src/image/image_file.rs:195-260`)。

resume(`sandbox.rs:1372`):rootfs 同 spawn 路径;内存按 mem image key **共享一个只读 ublk 内存设备**(`get_or_create_shared_mem`,`src/sandbox/ublk/device.rs:470-570`),FC File backend `load_snapshot_file` 恢复。再次 pause 时新的 dirty 层叠在继承的 mem lowers 上——**内存层链增量增长,超上限才 compact**(`overlaybd_snapshot.rs:560-588`)。uffd 懒加载路径已实现(`storage/uffd-core/`)但主链路未启用(`instance.rs:476-508` 标 dead_code)。

### 2.4 commit 与 fork

- `snapshot create` = pause + 立即 resume + 层去重入库(publish 到 managed-layers)+ P2P 广播(`src/snapshot/manager.rs:102-193`);实测 6.97 s(全量 artifact commit),此后 `aenv start <snapshot>` 只要 83 ms。
- `fork`(`src/orchestrator/service.rs:533-670`):源 pause → 立即 resume → N 个 child 并行 `from_snapshot_config` 复用同一份 pause artifact;child 只新建自己的可写 upper 与 FC 进程,不产生新 commit。

## 3. AgentEnv 实测(BareMetal,2026-08-20)

### 3.1 spawn(CoW from template)

| 场景 | 结果 |
|---|---|
| 顺序 spawn ×16(同 ubuntu 模板) | 31–66 ms/个(p50 ~38 ms);**磁盘增量共 524 KiB**(≈33 KiB/实例,只有 upper 元数据) |
| 16 并发 spawn | wall 89 ms(≈180 spawns/s) |
| warm 模板首次启动 | 74 ms |
| 每 FC 进程 RSS | 平均 ~10 MiB(stub 阶段);warm pool 64 个仅 0.1 GiB |

### 3.2 写路径(可写层)

| 场景 | 结果 |
|---|---|
| guest 内 100 MiB `conv=fsync` | `upper.data` = 100.0 MiB + 32 KiB 索引 → **写放大 ≈ 1.0** |
| 写吞吐(经 ublk 用户态块设备) | 213 MB/s |
| 未 fsync 的写 | 留在 guest page cache,upper.data 不增长(此时 fork 会连同内存一起拷走,文件在 fork 中仍可见) |

### 3.3 C/R(同 [前序文档](20260820T133500Z-baremetal-agentenv-substrate-deployment-cr-verification.md))

| 操作 | 延迟 |
|---|---|
| pause(checkpoint) | 62–143 ms;+512 MiB guest 脏页几乎不变(增量只含 dirty ranges) |
| resume(restore) | 80–110 ms |
| committed snapshot create | 6.97 s(一次全量入库) |
| restore from committed snapshot | 83 ms |
| 32 并发 pause / resume | wall 284 / 203 ms(113–158 ops/s) |

### 3.4 fork(实测语义与成本)

| 项 | 干净 sandbox | 100 MiB 脏(内存 page cache) | 100 MiB 脏(fsync 落 upper) |
|---|---|---|---|
| fork 延迟 | 192–196 ms | ~195 ms | 250 ms |
| 磁盘增量 | ~0 | mem_overlaybd 122 MiB | mem_overlaybd 231 MiB + inherited-layer 101 MiB |

产物结构(`firecracker-work/managed-snapshots/<src>/<fork>/`):

```text
mem_overlaybd/overlaybd.commit            # 父内存(guest RAM)压成的层
rootfs/inherited-layers/0000/snapshot.commit   # 父可写层封存为只读继承层
rootfs/{image.json, upper.data}           # fork 自己的新可写层
```

语义验证:fork 内可见父的全部内存+文件状态(big.bin/big2.bin/marker 均在);fork 内修改不影响父(块级隔离)✓。

**关键观察:fork 的磁盘成本 = O(父脏内存),一次性物化一份共享 mem 层**。~~初版本文曾记录"每个 fork 各自物化一份 mem 层(3 个 fork 各 122 MiB)"~~——2026-08-21 以 1GiB 脏复核(fork×3 du 增量 = 937 MiB ≈ 单份 915 MiB mem 层;fork×2 工件树仅一个 `mem_overlaybd/overlaybd.commit`),证实**同父所有 child 共享同一份 mem 层与封存 upper**,每个 child 只新建私有空 upper。仍与 AKernel FICLONE fanout 有本质差距:这里是线性物化真字节(写带宽付全价,0.9×S),AKernel 是元数据级 CoW(磁盘增量 0)。详见[后续评估文档](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md)§4.2。

## 4. AKernel C/R 实现(gVisor fork + filestore 领养)

### 4.1 机制

- 运行时:gVisor runsc(自研 fork,8 个 C/R 相关 commit);sentry 内 EROFS rootfs(mmap 共享宿主 page cache,`gvisor/pkg/erofs/erofs.go:222-231`)+ tmpfs upper overlay,upper 由宿主匿名 filestore 文件支撑(`gvisor/runsc/boot/vfs.go:647-760`)。
- C/R 不是 CRIU,是 **gVisor 原生 state checkpoint**(sentry 全状态序列化:任务/MemoryFile/fd/挂载树/netstack,`gvisor/pkg/sentry/control/state.go:23-35,264-283`)+ 自研零拷贝扩展:
  - `--skip-filestore-pages`:可写层私有 MemoryFile 只存 `ContentExternal` 段元数据,**页内容不进 checkpoint** → checkpoint 从 O(S) 变 O(1)(`gvisor/pkg/sentry/pgalloc/save_restore.go:122-143,274-320`);
  - `--filestore-snapshot-dir`:冻结窗口内对每个 filestore `ioctl(FICLONE)` 快照 + `filestores.json` manifest(尺寸 + 首尾 4K 采样 SHA-256)(`gvisor/pkg/sentry/kernel/filestore_snapshot.go:44-86`);
  - restore `--filestore-adopt-dir [--filestore-clone-on-adopt]`:按 ResourceID 领养宿主文件(校验指纹),clone-on-adopt 再 FICLONE 出私有 CoW 副本、模板保持不可变可多次复用(`gvisor/runsc/container/filestore_snapshot.go:158-266`)。
- 网络 restore 侧由 sandboxd 重建 veth/IP/ACL(statefile 不含宿主侧);跨节点有 CPUID 门禁 + 逃生口。
- 现状:开源 AKernel v0.1.0 **不含 C/R**(proto 无 Checkpoint,README 标 planned);完整能力活在 gvisor fork + sandboxd-pr16 + parkd 三处。

### 4.2 实测(cr-bench,已有数据)

| 项 | stock gVisor | AKernel 分支(FICLONE) |
|---|---|---|
| park @S=1G 可写层 | 1236 ms(1G 配额下退化 3854 ms) | **133 ms** |
| unpark @S=1G | 2195 ms | **244 ms** |
| checkpoint image 大小 | 1025 MB | **1 MB**(filestore 外置) |
| pgalloc SaveTo 256MB | 60.8 ms | 0.084 ms(ExternalContent) |
| pgalloc LoadFrom 256MB | 431.6 ms | 0.241 ms(领养) |
| 对 S / cgroup 配额的敏感性 | 强(512M 配额下差 28.8×) | **平坦**(126–144 ms / 232–273 ms) |
| fanout(N×fork) | — | N∈{1..64} 逐实例 195–290 ms 对 N/S 双无关,**df 增量 0**;64 实例风暴 wall 3.4 s |
| 内存态(1GiB urandom) | — | ckpt 1190 ms(1.2 ms/MB)、restore 470 ms(0.46 ms/MB) |
| 跨节点 | — | tar 口径 467 MB/s(O(S) 外显);restore@node2 1025 ms vs 同机 265 ms |

## 5. 对比分析

### 5.1 checkpoint/restore:增量 vs 零拷贝

- **AgentEnv pause 的成本模型是 O(dirty)**:KVM dirty tracking + process_vm_readv 只读脏页。稳态 C/R(每次脏一小部分)在 60–140 ms;但 fork(等同"从零开始的第一代快照")必须物化全部脏内存(同父 child 共享一份,2026-08-21 修正)。增量 mem 层链还会持续增长直到 compact,长生命周期 sandbox 的层链深度是隐形成本。
- **AKernel park 的成本模型是 O(1) + O(M)**:filestore FICLONE 对可写层大小不敏感(133 ms @1G),但 sentry 匿名内存仍是序列化(1.2 ms/MB save、0.46 ms/MB load)。AKernel 的 checkpoint image 只有 1 MB(filestore 外置为 sidecar),AgentEnv 的 vm_state.bin 也是 ~20 KB(state-only Diff)——**两边的"控制状态"都很小,差别全在可写层与内存的搬运方式**。
- 数字上:稳态 pause/resume AgentEnv(62–143/80–110 ms)优于 AKernel 分支(133/244 ms),但 AKernel 对 S 平坦——S=1G 时 AgentEnv 的 fork 类全量操作要物化 GB 级数据(实测 100 MiB 脏已 +231 MiB 磁盘、250 ms;线性外推 1 GiB 脏 ≈ 2.4 GiB 磁盘、秒级)。

### 5.2 spawn:两种共享

- AgentEnv:spawn 38 ms,只读层(managed-layers)内容寻址共享,每实例 33 KiB 元数据;磁盘成本与实例数几乎无关。**但 rootfs 块设备聚合发生在用户态 daemon**(LSMT 查层),每次读有 lower 栈查找开销;好在有 premerged-index 缓存缓解。
- AKernel:EROFS mmap 直接共享宿主 page cache,读路径是内核 mmap(零用户态开销);但 spawn 依赖 runsc create 全量起 sentry+gofer,无 warm pool(开源版),冷启动没有 AgentEnv 的 Firecracker pool(2 个 warm stub,启动 74 ms)与 ublk pool 快。
- 并发扩展:AgentEnv 16 并发 spawn 89 ms、32 并发 pause 284 ms,daemon 是单点但吞吐尚可;AKernel fanout N=64 wall 3.4 s(≈19/s 派生),FICLONE p95 1 ms,瓶颈在 restore 而非拷贝。

### 5.3 CoW 粒度与写放大

| | AgentEnv | AKernel |
|---|---|---|
| 运行时可写层 | log-structured upper(追加写 + 512B segment 索引) | sentry tmpfs → 宿主 filestore(4K 页) |
| 派生 CoW | 块级(继承层 + 新 upper) | 文件级 reflink(XFS extent) |
| 写放大实测 | ≈1.0(100MiB → 100MiB+32KiB) | 无拷贝(reflink);运行期写即写 filestore |
| 依赖 | 无特殊 FS 要求(任何本地 FS) | **必须 reflink=1(XFS)**;FICLONE 不可用时退化 FullCopy(实测该盘 EOPNOTSUPP 下 1.4–1.5 s) |

AgentEnv 的 512B 块粒度对"改一个字节"的派生场景更省(只加一个 segment);AKernel 的文件级 reflink 对"整文件级派生"更自然,且 CoW 由内核管理、无用户态索引维护。AKernel 的硬依赖是 reflink FS——FICLONE 不可用时 Snapshot 退化 FullCopy(~1.5 s),AgentEnv 无此依赖但多了一层用户态块设备(写吞吐 213 MB/s,纯块设备/local FS 会更高)。

### 5.4 fork 语义对比(核心分野)

| | AgentEnv fork | AKernel fanout(clone-on-adopt) |
|---|---|---|
| 源 | running sandbox(pause→resume 窗口) | checkpoint 模板(或 --leave-running 快照) |
| 延迟 | 195–250 ms/个 | 195–290 ms/个(对 N、S 均平坦) |
| 磁盘 | **O(脏内存+可写层)/fork,各自一份** | **0 增量**(extent 共享) |
| 内存 | fork 物化 mem_overlaybd 层 | 每次 restore 各自加载 O(M)(页不共享) |
| 隔离 | 块级(继承层只读 + 私 upper) | extent 级(FICLONE CoW) |
| 多代 | 未验证(层链继承应支持) | 已验证 gen3 血统封闭 + 旁支隔离 |

两者延迟相近,但**AKernel fanout 的磁盘成本是常数,AgentEnv fork 是线性的**——在"从同一基线大规模派生"(agent rollout/批量沙箱)场景 AKernel 优势随 N 与 S 线性放大(N=64×1G 时 AgentEnv 口径外推要物化 ~百 GiB,AKernel 实测 df 增量 0)。反向地,AgentEnv fork 从 running 实例直接派生且带完整 microVM 边界,AKernel 的 fork 本质是"从 checkpoint 派生",从 running 派生需先 leave-running checkpoint。

### 5.5 内存恢复策略

- AgentEnv:restore 用 FC File backend 全量映射 mem 层(共享只读 ublk 设备),**脏页跟踪为下一次增量服务**;uffd 按需分页路径已实现未启用(`storage/uffd-core`,512B segment 增量)——启用后可接近 AKernel ExternalContent 的懒恢复。
- AKernel:restore 反序列化 O(M)(1GiB ~470 ms),无懒加载;但 pgalloc 领养路径(0.24 ms @256MB)证明**若内存也走 External + 领养/uffd,两边会收敛到同一设计点**。这是两边互相借鉴最明显的位置。

### 5.6 生态与工程面

| | AgentEnv | AKernel |
|---|---|---|
| 镜像格式 | OverlayBD(OCI 兼容,支持 native/standard 双路径,registry 按需取层) | EROFS 镜像 + OCI(经 distill-fs/Nydus 按需读) |
| 后端存储 | posix_fs / OSS/S3,P2P 分发 | 本地 XFS(reflink 硬需求);跨节点 O(S) tar |
| 分布式 | prototype(Gateway/Scheduler,见前序 survey) | parkd 原型;跨节点已有 CPUID 门禁 |
| 开源完整度 | C/R/fork/pool 全部可用 | v0.1.0 无 C/R;完整能力在 fork 仓库 |

## 6. 对 AKernel 的启示

1. **增量内存层是 AgentEnv 最值得吸收的机制**:KVM dirty tracking + 按脏页压层 + 层链继承,使稳态 C/R 只付脏页代价。AKernel 目前只有整点快照,可考虑对 MemoryFile 引入同样的 dirty-range 增量链(filestore 已有 FICLONE,内存侧加增量层即可),代价是要处理层链 compact。
2. **uffd 懒恢复应尽快启用**(AgentEnv 已写好未接;AKernel 可用同一思路):把 restore 从 O(M) 变 O(fault-on-demand),直接消掉 1GiB ~470 ms 的固定成本——对 unpark 延迟(244 ms)是下一个最大头。
3. **AKernel 的常数成本 fork 是护城河,但硬依赖 reflink FS**:应在 FICLONE 不可用时自动探测并降级(AgentEnv 式物化拷贝),而不是 1.5 s FullCopy;反向 AgentEnv 的 fork 磁盘线性成本在大规模派生场景不可持续,值得评估 mem 层跨 fork 内容寻址共享(同父同脏集的 fork 可共享 mem 层,代价是要处理写时复制或引用计数)。
4. **写路径**:AgentEnv 用户态 ublk(213 MB/s)是可接受的代价换来了 pool/按需下载/下载门控;AKernel 的 mmap rootfs 读路径更优,两者可组合(EROFS 只读层 + reflink filestore 可写层已是这个方向)。
5. **度量口径**:后续对比实验应固定同一负载(同 dirty 集、同 S)下测:稳态 pause/resume 循环、首代 fork、N-way fanout、层链增长随代数的曲线——本次两边数据来自不同负载,绝对值只做定性对照。

## 7. 证据索引

### AgentEnv(均为 BareMetal `/root/AgentEnvWorkSpace/AgentENV` 下路径)

- 内存 Diff 快照 + dirty ranges + process_vm_readv:`src/sandbox/firecracker/instance.rs:457-474`、`src/sandbox/firecracker/overlaybd_snapshot.rs:42,759-843`、`src/sandbox/firecracker/process_vm_reader.rs:10-25`
- mem 层链增量继承与 compact:`src/sandbox/firecracker/overlaybd_snapshot.rs:560-588`
- rootfs restack 封存:`src/sandbox/firecracker/sandbox.rs:746-753`、`storage/overlaybd/src/image/image_file.rs:195-260`
- 运行时 upper 物化(512B LSMT):`storage/ublk-daemon/src/runtime.rs:124-255`、`storage/overlaybd/src/lsmt/index.rs:8-16`
- resume 共享内存设备:`src/sandbox/ublk/device.rs:470-570`
- fork 编排:`src/orchestrator/service.rs:533-670`、backend `src/sandbox/firecracker/sandbox.rs:340-394`
- commit/publish 与 managed-layers:`src/snapshot/manager.rs:102-193`、`src/snapshot/repository/backends/posixfs/artifacts.rs:44-475`
- 下载门控/后台下载:`storage/overlaybd/src/download_gate.rs:1-60`、`src/cfg.rs:469-507`
- uffd 未启用:`src/sandbox/firecracker/instance.rs:476-508`(dead_code)、`storage/uffd-core/src/handler.rs`
- warm pool 配置:`src/cfg.rs:774-818`

### AKernel(本地工作区)

- runsc 默认运行时与 spawn 路径:`AKernel/src/sandboxd/internal/server/server.go:986-1093`、`pkg/runtime/runsc/client.go:112-233`
- EROFS rootfs + overlay/filestore:`gvisor/runsc/boot/vfs.go:570-760`、`gvisor/runsc/boot/mount_hints.go:344-373`
- skip-filestore-pages / ExternalContent:`gvisor/pkg/sentry/pgalloc/save_restore.go:122-143,274-320`
- FICLONE 快照 + manifest:`gvisor/pkg/sentry/kernel/filestore_snapshot.go:44-86`
- 领养 / clone-on-adopt:`gvisor/runsc/container/filestore_snapshot.go:158-266`
- 匿名 filestore 创建:`gvisor/runsc/container/container.go:1257-1283`
- XFS reflink 要求:`AKernel/src/sandboxd/pkg/volumemanager/xfs.go:46,73`、`config/config.go:92-102`
- parkd fanout:`AKernel_scheduler/prototype/cr-bench/parkd.py:33-36`
- 性能矩阵:`cr-bench/e1/bench.log`、`cr-bench/e2/results/`、`AKernel_Docs/Surveys` 同目录 `docs/CR/status/20260820T0200Z/0330Z/0655Z` 系列
- 开源 v0.1.0 无 C/R:`AKernel/README.md:37,206`、`AKernel/src/sandboxd/api/runtime/v1/sandbox-api.proto:24-36`;pr16 版:`sandboxd-pr16/api/runtime/v1/sandbox-api.proto:26-39`
