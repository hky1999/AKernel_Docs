# C/R 与 guest 内存大小的关系:Firecracker / AgentEnv / gVisor / CubeSandbox 四方对照 —— 匿名内存线 vs 可写层线 Survey

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T023535Z`
> 缘起:有同行提出"Firecracker 的 C/R 时间和镜像大小随 guest 内存线性增长"。本文从源码 + 同机实测两个层面系统回答这个问题,并澄清一个此前文档的盲区:**我们过去测的主要是可写层(RWFS/filestore)线,guest 匿名内存线本次才首次实测**。
> 对象与证据:
> - **Firecracker 上游 1.16.1**:本地 shallow clone `/tmp/fc-src`(源码级分析);
> - **AgentEnv**(commit `7f4a9b9` + 私有 FC fork `kvcache-ai/firecracker v1.15.1-patch`):BareMetal 源码 + 2026-08-24 新实验;
> - **gVisor fork**(PR#14228,HEAD `1985ab80b`):本地源码 + BareMetal 新实验(`/root/cr-bench-bm/scripts/anon_scale.sh`);
> - **CubeSandbox**(Tencent,`/root/CubeSandboxWorkSpace/CubeSandbox`):BareMetal 源码级分析(未实测)。
> 实测环境:BareMetal(104 vCPU / 187 GiB / Ubuntu 24.04 / kernel 6.8,云盘),与 [PR 评估 survey](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md) 同机。
> 前序:[BareMetal 部署验证](20260820T133500Z-baremetal-agentenv-substrate-deployment-cr-verification.md)、[OverlayBD vs AKernel](20260820T163100Z-agentenv-overlaybd-spawn-cr-cow-vs-akernel-survey.md)、[RWFS×ROFS 演进](20260821T125008Z-rwfs-rofs-deltabox-mechanism-evolution-agent-infra-cr-fork-survey.md)

---

## 1. 结论摘要

**同行的说法对"原版 Firecracker 的 Full snapshot"完全正确,但它不是普适规律——正确的统一框架是把 C/R 成本拆成三条线:**

| 成本线 | 定义 | stock Firecracker | AgentEnv(FC fork) | runsc(PR#14228) | CubeSandbox |
|---|---|---|---|---|---|
| **① 可写层线(RWFS)** | guest 文件系统写入量 S | 无独立可写层(rootfs 即内存盘) | O(upper 脏) 封存 rename,近 O(1) | **O(1)**(FICLONE 外置) | O(1)(CubeCoW reflink) |
| **② 匿名内存线(RSS)** | guest 匿名页(应用堆/栈/MAP_ANON、tmpfs) | Full: **O(guest RAM)** | O(匿名脏) | **O(RSS)**(未解决) | Incremental: O(自 restore 匿名页);SoftDirty: O(窗口脏) |
| **③ 脏集线(dirty)** | 自上次快照以来写过的页 | Diff: O(dirty) | O(dirty ∪ 读回页)† | (并入①②) | SoftDirty: O(窗口脏) |
| **restore** | 灌回成本 | **≈O(1)**(mmap 懒) | ≈O(1)(mmap 懒) | O(1)+首触冷读;匿名线 O(RSS) | ≈O(1)(mmap 懒) |
| **镜像大小** | | Full: 恒=mem_size;Diff: 物理 O(dirty) | mem 层 O(dirty)+封存 upper | image ~1MB+FICLONE;**匿名线 O(RSS)** | base FICLONE+delta O(脏) |

† AgentEnv 特有:用户盘 Async IO 引擎对**读完成也标脏**,文件页缓存被 DAMON 回收后任何重读都会整批重新物化(§4.3,实测复现)。

核心事实五条:

1. **stock Firecracker Full snapshot:时间与镜像都精确 O(guest RAM)**——逐 slot 全量顺序写、无零页跳过、还会把全部 guest 内存 fault in(源码 + 官方文档原话,§3)。同行的说法在这个口径下成立;且 restore 端反而是 mmap 懒加载近 O(1),"慢在 create、快在 resume"。
2. **Diff 路线把基数从 RAM 缩到脏页**:FC 上游 Diff = KVM dirty log(物理大小 O(脏)×4K,逻辑大小恒=mem_size);CubeSandbox 更进一步用 soft-dirty 缩到"窗口脏"。**"线性于 RAM"只在全量口径下成立**。
3. **gVisor 在 PR#14228 之后只剩匿名线是 O(RSS)**——本次首次实测(64M→4G 完美线性:ckpt ≈20ms+0.64ms/MB、restore ≈100ms+0.23ms/MB、image=M+1.5MB);同一系统内同量写 filestore 则 54ms/0.6MB(O(1))。两条线的源码分叉点:filestore 有宿主文件身份(可 ContentExternal/FICLONE),匿名页住 sentry memfd(无身份,必须逐页序列化)。
4. **AgentEnv 的 pause 是 O(脏) 且"读也计脏"**:实测两种脏法首停线性(shm 128/256M、file 256/512/1024M:~0.65ms/MB),但用户盘 async 引擎读完成即标脏 → 恢复后重读大文件会把整批页重新物化(实测:pause1=817ms → 无 exec pause2=57ms → md5sum 后 pause3=826ms)。这解释了此前两份文档表面矛盾的 数据。
5. **CubeSandbox 是第三条路线的代表**:快照成本 O(窗口脏)、恢复成本 O(1)(MAP_PRIVATE mmap)、代际共享靠 XFS FICLONE——与 FC(全量)/gVisor(逐页序列化)都不同;代价是 soft-dirty 的 `clear_refs` 重置本身 O(全部 PTE),被摊入首次快照。

对"为什么之前的文档里没看到类似说法"的回答:此前所有实测的 S 维度(64M–4G)都是**可写层线**(filestore/overlaybd),正是各系统优化最充分的一条线;guest 匿名内存线在 gVisor 侧从未单独测过(且 163100Z survey 把 e2 的"1GiB urandom"误标为内存态,实际写的是 /bench.bin 可写层,本文 §5.3 修正),在 AgentEnv 侧只测过小脏集稳态。盲区即在此。

---

## 2. 三线成本模型(分析框架)

一次 C/R 的全部数据运动可分解为:

```
checkpoint = 可写层(S) + 匿名内存(RSS) + 自上次以来的增量(dirty) + 元数据(设备/CPU/任务状态)
restore   = 上述各线的灌回(逐页 | mmap 懒 | FICLONE 领养)
```

- **线①(可写层)**:guest 文件系统的持久数据。在 gVisor 里是 overlay upper 的 filestore;AgentEnv 是 overlaybd upper;CubeSandbox 是 CubeCoW 卷。三者都已做到 O(1)(reflink/rename 语义)。
- **线②(匿名内存)**:无文件背景的页——应用 heap/stack、MAP_ANON、tmpfs(/dev/shm)。**这是 microVM 世界的"guest RAM"、gVisor 世界的 sentry MemoryFile**。本文焦点。
- **线③(脏集)**:增量快照的记账粒度,只在"diff 型"系统里有意义。KVM dirty log(x86 EPT/PML)、soft-dirty(pagemap bit55)、mincore 近似是三种实现。

关键不对称:**checkpoint 的时间/空间可以是 O(脏),但首次快照的脏=全部已写页,逼近 O(RAM)**;restore 若是逐页灌回则 O(镜像),若是 mmap 懒加载则 O(1)+首触。所以"C/R 随内存线性"必须分成 (a) 哪条线 (b) 全量还是增量 (c) create 还是 restore 三个问题分别回答。

---

## 3. Firecracker 上游(1.16.1):Full 线性、Diff 脏页、restore 懒

源码证据(`/tmp/fc-src`):

### 3.1 Full snapshot:精确 O(guest RAM)

`KvmVm::snapshot_memory_to_file`(`src/vmm/src/vstate/vm.rs:574-634`):

```rust
// vm.rs:592-593  文件长度恒定 = guest 内存大小
let expected_size = mem_size_mib(self.guest_memory()) * 1024 * 1024;
// vm.rs:618-628
match snapshot_type {
    SnapshotType::Diff => { let b = self.get_dirty_bitmap()?;
                            self.guest_memory().dump_dirty(&mut file, &b)?; }
    SnapshotType::Full => { self.guest_memory().dump(&mut file)?; ... }
};
```

- Full 走 `GuestMemoryMmap::dump`(`vstate/memory.rs:1031-1044`):每个 plugged slot `write_all_volatile` **全量顺序写,无脏页判定、无零页跳过**;unplugged slot 只 seek(稀疏洞)。
- 官方文档原话(`docs/snapshotting/snapshot-support.md:143-147`):"Taking a full snapshot of a microVM results in the full contents of guest memory being written to the snapshot, and particularly, in **all guest memory being faulted in**"——即 Full 快照不仅写 O(RAM) 字节,还会把整个 guest 内存拉热(resident),代价双重。
- 磁盘供给约束也是显式的(`snapshot-support.md:512-514`):"Depending on VM memory size, snapshots can consume a lot of disk space. Firecracker integrators **must** ensure that the provisioned disk space is sufficient"。

### 3.2 Diff snapshot:物理 O(脏页)、逻辑恒 = mem_size

- 脏页判定 = `KVM_GET_DIRTY_LOG`(取走即清)∪ FC 内部 `AtomicBitmap`(捕捉 vMMIO 等宿主侧写入,`memory.rs:460-478`)。
- 干净页只 `seek` 跳过(`memory.rs:482-487`)→ diff 文件是**稀疏文件,物理块 ≈ 脏页数 × 4KiB**,但逻辑大小仍 `set_len(mem_size)`(`vm.rs:614-616`)。
- dirty tracking 在 **guest 内存创建时**由 `track_dirty_pages` 决定(`memory.rs:364-367,423-429`);未启用时 diff 走 mincore 近似(resident 页全算脏,`vm.rs:739-770`,文档要求关 swap)。
- Diff 成功后清 FC bitmap(`memory.rs:1076`);写文件失败则把 KVM bitmap OR 回内部 bitmap 不丢脏页(`memory.rs:1073-1077`)。
- **第一次 diff ≈ 全量**(自 boot/restore 以来所有脏页;`memory.rs:1560` 测试注释)。

### 3.3 restore:File 后端是 mmap,近 O(1)

`guest_memory_from_file`(`persist.rs:516-526`)→ `memory::snapshot_file`(`memory.rs:902-928`)→ **`MAP_PRIVATE|MAP_FIXED` 把 memory image 直接映射为 guest RAM**(`memory.rs:352-362`)。文档原话(`snapshot-support.md:79-84`):"instead of copying contents from file to memory, Firecracker creates a memory file, resulting in runtime on-demand loading of memory pages"。另有 Uffd 后端(`persist.rs:543-575`)把缺页交给外部 handler。**即 stock FC 的 C/R 慢在 create(O(RAM) 全量写),resume 本身近 O(1)**。mem size 不能在 load 时改(从 state 读出并强校验 image ≥ region 总和,`memory.rs:908-919`)。

**小结:同行说法对 stock FC 的准确度——Full snapshot 时间/大小 O(guest RAM) ✅(且外加全内存 fault-in);但 restore 是懒加载 ✅ 不线性;Diff snapshot 物理 O(脏) 不线性。**

---

## 4. AgentEnv(kvcache-ai FC fork + OverlayBD):O(脏) 但"读也计脏"

### 4.1 机制(源码:BareMetal AgentENV + FC fork `v1.15.1-patch`)

pause 链(`src/sandbox/firecracker/sandbox.rs:660-679,831-837`):

1. `PATCH /vm {Paused}`;
2. `PUT /snapshot/create` **Diff + 不传 mem_file_path(state-only)**——只落 `vm_state.bin`(~20KB vCPU/设备状态),FC 侧不 dump 内存(`instance.rs:457-464`;FC fork persist.rs:178 `if let Some(mem_file_path)` 才 dump);
3. `GET /vm/dirty-memory-ranges`(**FC fork 私有 API,上游不存在**):KVM log 读后即清但位被 OR 回 FC 内部 bitmap,返回 = KVM ∪ 内部位图,逻辑非清除(FC fork `vstate/vm.rs:329-341`);
4. `process_vm_readv` 按 HVA 直读 FC 进程脏页 → 512B 扇区 SegmentMapping → overlaybd commit 层(`overlaybd_snapshot.rs:717-784`),默认不压缩(`src/cfg.rs:456-463`);
5. rootfs upper 同窗口 close-seal + rename 封存(`sandbox.rs:741-756`)。

resume:mem 层栈挂成 shared-mem ublk 设备 → FC fork `snapshot_file()` **MAP_PRIVATE mmap 惰性缺页**(FC fork memory.rs:646-672)。层链 >32 才压缩合并(`overlaybd_snapshot.rs:37`)。

### 4.2 实测(2026-08-24,模板沙箱 2vCPU/1024MiB,`/root/AgentEnvWorkSpace/aenv_anon_scale.sh`)

| 变体 | MB | pause#1 ms | pause#2 ms† | resume ms | pause#1 新增工件字节 | pause#2 新增 |
|---|---:|---:|---:|---:|---:|---:|
| shm(/dev/shm,匿名) | 128 | 175 | 56 | 74 | 143 MB(≈1.1×M) | 3.8 MB |
| shm | 256 | 269 | 57 | 111 | 281 MB | 2.8 MB |
| shm | 512 | 471 | 52 | 90 | 533 MB | 2.9 MB ‡ |
| file(/root,页缓存) | 256 | 260 | 54 | 104 | 558 MB(**≈2×M**) | 3.0 MB |
| file | 512 | 477 | 53 | 99 | 1108 MB | 3.2 MB |
| file | 1024 | 710 | 58 | 113 | 2032 MB | 3.0 MB |

† pause#2 前无任何 exec。‡ 该行 md5 假阴性:guest `/dev/shm` 是 tmpfs 默认 50%RAM=512MiB 上限,512M dd 触顶失败 → `&&` 短路 want 为空;单独复测(分层 exec)**md5 MATCH,匿名内容恢复正确**。

读法:

- **pause#1 两种脏法都线性** ≈45ms + 0.65ms/MB;**pause#2 平坦 52–58ms**——脏集在新 FC 进程(resume 换进程,dirty 从零)中确实重置;
- **file 变体工件 ≈2×M**:mem 层(页缓存脏页 M)+ rootfs upper 封存(文件数据 M)——同一份数据在两条线上各记一次;shm 只有 mem 层 ≈1.1×M;
- 对照 041720Z 的 file-1G 三轮 824–1305ms(每轮前有 md5sum exec)与 133500Z 的 shm-512M 仅首轮 257ms(轮间只 touch 计数文件):**矛盾由"读也计脏"统一解释**(下节)。

### 4.3 "读也计脏":O(脏) 语义的隐性放大器(源码 + 因果实验)

三个源码事实合力(FC fork + AgentENV):

1. guest 默认 boot args 开 DAMON 回收且 `damon_reclaim.skip_anon=Y`——**只回收 file-backed 页**(`src/sandbox/firecracker/config.rs:30-46`):/root 的页缓存会被回收腾内存,/dev/shm 匿名页不回收;
2. 用户盘 `IoEngine::Async`,FC fork 的 async 引擎在**每个完成的块请求(读和写都)上 `mark_dirty`**(FC fork `devices/virtio/block/virtio/io/async_io.rs:89-117`,esp. 115)——目的是让设备 DMA 写入的页能进 diff,代价是**重读也会整批标脏**;
3. `dirty-memory-ranges` 返回的是 FC 内部 bitmap 累积位(非清除),下一次 pause 全部物化。

因果实验(2026-08-24,`redirty_test.sh`,file-1GiB,单沙箱三轮):

```
pause1 = 817 ms   (dd 1GiB 脏)
pause2 =  57 ms   (resume 后无任何 exec)
pause3 = 826 ms   (resume 后 exec 了一次 md5sum /root/f.bin)
```

**含义**:AgentEnv 的 pause 稳态成本不是"写脏"而是"**触摸过(含读)且可能被回收重读的页**"。对 agent 负载是实际风险——恢复后重读大 repo/大文件的每一步都会把下一次 pause 推回 O(读集)。gVisor 匿名线无此问题(只按写序列化),但 gVisor 有自己的 O(RSS) 常态成本(§5)。

---

## 5. gVisor/runsc(PR#14228):两条线的源码分叉与首次匿名线实测

### 5.1 源码:为什么 filestore 能 O(1) 而匿名内存不能

gVisor 里 guest 的"内存"由 sentry 的 `pgalloc.MemoryFile` 承载,但有**两个实例、两种后端**:

| | 匿名线 `k.mf` | filestore 线(私有 MF) |
|---|---|---|
| 承载 | 宿主 **memfd**(`runsc/boot/loader.go:1091-1097`,只有 fd 无路径) | gofer mount ns 里的**宿主命名文件**(create-then-unlink,`runsc/container/container.go:1246-1266`) |
| 内容 | 应用 heap/stack/MAP_ANON + 沙箱内 tmpfs(/dev/shm、/tmp)(`pkg/sentry/fsimpl/tmpfs/tmpfs.go:227-257`;`mm/pma.go:334-345`) | overlay upper 文件数据(sentry 直写宿主文件页缓存,**不经 gofer 数据面**,`boot/vfs.go:773-779`) |
| checkpoint | **逐页扫描+去零+序列化**(O(RSS)):`pgalloc/save_restore.go:593-691`(零页 `bytes.Equal` 逐页比对,非零段 `w.Write(s)` 原始字节) | `ContentExternal` 只写元数据+指纹(:341-382),页内容不进 image |
| 为何不能外置 | memfd 无宿主文件身份;restore 要求 `AdoptExistingFile`(:1233-1235);memfd 在 shmem 上**不可 FICLONE** | 有身份:ResourceID + 宿主文件 → FICLONE 快照(:290-296)+ adopt 校验 |
| 谁设置 | `k.mf.SaveTo` 永远不带 ExternalContent(`kernel/kernel.go:913-923` 匿名 MF 先存、私有 MF 后存且只给后者设 External) | `runsc/sandbox/sandbox.go:1661-1691` |

补充:压缩作用于 statefile wire 层且**与 pages 文件互斥**(`sandbox.go:1827-1847`)——开 flate 则匿名页内联进压缩流;`--compression=none` 才有独立 pages 文件(restore 可 `AsyncPagesFileLoad` 后台并行加载,字节数仍 O(RSS),`save_restore.go:1432+`)。去零是页粒度:全零段降级为 `KnownCommitted=false` 元数据,restore 靠 memfd 缺零页语义还原(:598-613)。

### 5.2 实测(2026-08-24,BareMetal,PR build,`--compression=none`,宿主 `sync` 后计时)

匿名线:`/dev/shm/a.bin` dd urandom(sentry tmpfs → k.mf);对照组:同量写 `/bench.bin`(overlay upper → filestore)。全新沙箱每格,双向 md5 全绿:

| cell | ckpt ms | image 字节 | FICLONE 工件(逻辑) | restore ms | md5 |
|---|---:|---:|---:|---:|---|
| anon 64M | 66 | 67.7 MB | 1 GiB chunk | 128 | ✓ |
| anon 256M | 202 | 269 MB | 1 GiB | 166 | ✓ |
| anon 1024M | **724** | **1.074 GB** | 1 GiB | **326** | ✓ |
| anon 2048M | 1433 | 2.147 GB | 1 GiB | 548 | ✓ |
| anon 4096M | **2624** | **4.296 GB** | 1 GiB | **1041** | ✓ |
| **file 1024M(对照)** | **54** | **0.57 MB** | 2 GiB(2 chunks) | **119** | ✓ |
| zero 2048M(零页) | 549 | **0.57 MB** | 1 GiB | 111 | ✓ |

拟合:匿名线 ckpt ≈ **20ms + 0.64ms/MB**、restore ≈ **100ms + 0.23ms/MB**、image = **M + 1.5MB**(urandom 不可压;`compression=none`)——三量全部严格线性。零页:去零生效(image 平)但**扫描仍付 O(M)**(549ms@2G ≈ 0.26ms/MB 纯扫描)。

**同一系统内的 A/B 直接给出结论**:1GiB 写匿名 724ms/1.07GB vs 写 filestore 54ms/0.6MB——PR#14228 把线①做到 O(1) 之后,**线②(匿名 RSS)成为 gVisor C/R 唯一剩余的线性项**。真实 agent(RSS 峰值 1.7G 实测)的滚动 ckpt 间隔下限正是由这条线决定的(0900Z"内存线 O(RSS)"的实测版)。

实验附带的两个边角发现:

1. **零写入沙箱无法 adopt**:可写层从未写入时 FICLONE 工件为空,restore 报 `filestore artifact ... is empty; it must contain the adopted writable-layer contents`——125008Z D4"零写入跳过"演进项必须处理此边角(空工件白名单或最小 marker);
2. 无宿主 sync 时 file 线 ckpt 从 54ms 退化为 9.3s(FICLONE 等待 1G 脏页写回)——再次确认 pr_eval 的 P3/P4 分离口径:写回是持久化选择,不是机制代价。

### 5.3 对既有文档的两处修正

1. **163100Z §4.2 的"内存态(1GiB urandom) ckpt 1190ms"标注有误**:cr-bench e2 的负载是 `dd of=/bench.bin`(`run_e2.sh:63-71`)——**filestore 线**(stock 全序列化口径),不是匿名线。匿名线的正确数字是本文 §5.2(724ms@1G,同机不同日,量级一致)。
2. **041720Z §4.1 的解读补全**:当时记录"每轮 pause 把全部脏页物化进 mem 层"——现在知道三轮都慢的直接原因是**轮间的 md5sum exec 重新标脏**(§4.3),不是 dirty 不重置;无 exec 时 pause#2 平坦 52–58ms(§4.2)。

---

## 6. CubeSandbox:第三条路线(源码级,未实测)

Cloud Hypervisor fork + KVM,VMM 侧三种快照模式(`hypervisor/vm-migration/src/lib.rs:110-123`):

| 模式 | 保存集合 | 复杂度 | 机制 |
|---|---|---|---|
| Full | 全部 | **O(guest RAM)** | 逐 range 写文件(`memory_manager.rs:3120-3128`) |
| Incremental | present ∧ anonymous | **O(自 restore 写过的匿名页)**,单调增长逼近 Full | `/proc/self/pagemap`+`kpageflags`(`pagemap_anon.rs:4-17`);对 reflink 克隆的 base **原地覆写**匿名页,产物仍是全量布局镜像(`memory_manager.rs:2372-2455`) |
| SoftDirty | anon ∧ bit55 | **O(本窗口脏页)** 真 delta | `clear_refs(4)` 布防 + pagemap bit55(`soft_dirty.rs:8-51,267+`;需 `CONFIG_MEM_SOFT_DIRTY=y`,否则静默降级 Incremental,`memory_manager.rs:2501-2530`) |

- **restore ≈ O(1)**:fast restore 把快照文件 fd `MAP_PRIVATE` mmap 为 guest RAM,只建 VMA(`memory_manager.rs:1326-1376,1509-1516`),写时内核 CoW——与 FC File 后端同构;
- **代际共享 O(1)**:内存镜像本体是 CubeCoW(XFS FICLONE)卷,Cubelet 快照前 reflink-clone 旧 base,增量只覆写匿名页(`Cubelet/services/cubebox/appsnapshot.go:570-583`);clone(n) = 快照 + n×create-from-snapshot,内存物理页靠 MAP_PRIVATE CoW 共享(deep-dive §4.2–4.3)——与 PR#14228 的 clone-on-adopt 同一模式;
- **代价**:`clear_refs(4)` 要在 mmap_lock 下遍历全部 PTE,是 multi-GiB guest 上 pause 时间的主导项,故采用惰性布防摊入首次快照(`memory_manager.rs:2475-2479`);Incremental 模式的脏集单调增长(deep-dive 实例:200MiB→15GiB 逐次递增),长寿命沙箱必须周期性 Full 重置;
- **宿主耦合澄清**:soft-dirty 只需内核配置;换 OpenCloudOS PVM 内核仅是**无 `/dev/kvm` 环境**的可选路径(`docs/guide/pvm-deploy.md:24-38`),有 `/dev/kvm` 的机器用主线 KVM 即可——此前 1100Z 文档"CubeSandbox 要换 KVM host 内核"的说法需要按此收窄。

**CubeSandbox 与 AgentEnv 的对照最有启发性**:两者同为 microVM + mmap 懒恢复 + reflink 代际,差异在脏集记账——AgentEnv 用 KVM dirty log + VMM 内部位图(读也计脏),CubeSandbox 用宿主进程 pagemap/soft-dirty(对 VMM 地址空间的匿名页记账,读不计脏、但 Incremental 单调增长)。**"线性于内存"在两家都不成立,成立的都是"线性于各自的脏集定义"**。

---

## 7. 四方总表与对同行说法的回答

### 7.1 成本模型总表(源码 + 实测)

| | FC 1.16.1 stock | AgentEnv(FC fork) | runsc PR#14228 | CubeSandbox |
|---|---|---|---|---|
| checkpoint 时间 | Full **O(RAM)**(全量写+全内存 fault-in);Diff O(dirty) | **O(脏 ∪ 读回页)** ≈45ms+0.65ms/MB(实测) | 线① **O(1)**(54ms@1G);线② **O(RSS)** =20ms+0.64ms/MB(实测);写回 O(dirty) 可剥离 | Full O(RAM);Incr O(anon 累积);SoftDirty **O(窗口脏)** |
| 镜像/工件大小 | Full **恒=mem_size**;Diff 逻辑=mem_size 物理=O(脏)×4K | mem 层 O(脏) + rootfs 封存 O(upper) → file 变体 **≈2×M**(实测) | image **~1MB**+FICLONE 工件(chunk 粒度逻辑大小);线② image=**RSS+1.5MB**(实测) | base(reflink 共享)+delta O(脏) |
| restore | **≈O(1)** mmap 懒 | ≈O(1) mmap 懒 + 首触(读回的页也计下次脏) | 线① O(1)+首触冷读;线② O(RSS) 逐页(0.23ms/MB 实测;pages-file 可后台并行) | ≈O(1) mmap 懒 |
| 增量语义 | KVM dirty log(取走即清) | fork 内部位图累积+进程切换重置 | 无增量(每次全量序列化线②) | soft-dirty 真增量 / pagemap-anon 半增量 |
| 依赖 | KVM;diff 需 dirty tracking 或关 swap 的 mincore | 定制 FC fork(私有 API)+ privileged + ublk | XFS reflink(filestore 线) | XFS reflink + `CONFIG_MEM_SOFT_DIRTY`(可选) |

### 7.2 对同行说法的精确回答

> "Firecracker 的 C/R 的时间和镜像大小随 guest 内存线性增长"

1. **对 stock FC Full snapshot:正确**,且比说法更糟(全内存 fault-in + 磁盘供给必须按 mem_size 预留,官方文档明示)。**但 restore 恰相反**:File 后端 mmap 懒加载,近 O(1),线性只发生在 create 侧。
2. **对 Diff snapshot:不成立**——物理大小与时间都是 O(脏页),首次快照才逼近 O(RAM)。
3. **推广到其它系统要分线**:AgentEnv 与 CubeSandbox 都把线①做成 O(1)、线②做成 O(各自脏集);gVisor PR#14228 把线①做成 O(1) 但**线②仍是 O(RSS) 全量**(无增量机制)——gVisor 的"线性项"不在可写层而在匿名内存,方向恰好与 microVM 系相反。
4. **为什么我们之前的文档没有类似数据**:此前实测的 S 都是可写层线;AgentEnv 只测了小脏集稳态;gVisor 匿名线首次在本文实测(并修正了 163100Z 的一处误标)。

### 7.3 各系统的真实"线性剩余"

| 系统 | 线性剩余 | 触发条件 |
|---|---|---|
| FC stock | Full: 全部 RAM | 每次 Full 快照 |
| AgentEnv | 首次/读回后的脏集 | 大写入后首停;**恢复后重读大文件**(读也计脏) |
| runsc PR#14228 | **匿名 RSS** | 每次滚动 ckpt(无增量),0.64ms/MB save + 0.23ms/MB load + 1B/B image |
| CubeSandbox | Incremental 单调增长的匿名集;SoftDirty 仅窗口脏 | 长寿命沙箱 Incremental 需周期 Full 重置 |

---

## 8. 对 AKernel 的启示(衔接 125008Z 演进项)

1. **匿名线是最后一条 O(RSS) 线,也是滚动 ckpt 间隔的物理下界**(G1 裁决的实测注脚)。三个可选的演进方向(按工程量排序):
   - **restore 侧先做**:`AsyncPagesFileLoad` 已存在(后台并行灌页),把它产品化为"懒恢复 + LLM 等待窗预热"——与 125008Z D3 async-warm 同一窗口调度器,恢复关键路径先降;
   - **增量序列化(中期)**:CubeSandbox 思路移植——k.mf 的 mmap 在 **sentry 进程地址空间**里,`/proc/<sentry-pid>/pagemap`+soft-dirty 理论上可用来只序列化"自上次 ckpt 以来写过的匿名页",base 镜像复用(正确性要求:restore 时 base+delta 分层灌回)。这把线②从 O(RSS) 降到 O(窗口脏),直接对齐 AgentEnv/CubeSandbox 的稳态语义;
   - **外置化(远期)**:给 k.mf 换 XFS 命名文件后端(获得 FICLONE 资格)——改动 sentry 内存分配核心路径,风险高,仅当增量路线不够时评估。
2. **零写入跳过(D4)的正确性边角**:空 filestore 工件被 adopt 拒绝(§5.2),实现时要么白名单空工件(校验 manifest 声明 size=0),要么注入最小 marker;
3. **对 AgentEnv 的对位优势话术**:AKernel 匿名线虽是 O(RSS) 全量,但**只按写序列化**(读不计脏)且 image 是自洽单文件;AgentEnv 读密集负载(恢复后扫 repo)会反复重物化 mem 层——agent 负载恰好是读密集的;
4. **度量口径沉淀**:今后 C/R 实验应固定三线分报——S(可写层)、RSS(匿名)、dirty(窗口脏),并注明 checkpoint/restore/首触三个位置;本文的 `anon_scale.sh` / `aenv_anon_scale.sh` 可直接复用。

---

## 9. 证据索引

### Firecracker 上游(`/tmp/fc-src`,1.16.1)

- Full/Diff 分派与文件定长:`src/vmm/src/vstate/vm.rs:574-634`(esp. 592-593,618-628)
- 全量 dump:`src/vmm/src/vstate/memory.rs:1031-1044`;脏页 dump 与 bitmap 融合:`memory.rs:442-518,1047-1080`;mincore 回退:`vm.rs:739-770`
- dirty tracking 生命周期:`memory.rs:364-367,423-429`;`src/vmm/src/resources.rs:506-512`;`src/firecracker/src/api_server/request/snapshot.rs:109-110`
- restore mmap:`src/vmm/src/persist.rs:516-526`;`memory.rs:352-362,902-928`;Uffd:`persist.rs:543-575`
- 文档:`docs/snapshotting/snapshot-support.md:79-84,110-111,143-147,209-214,283-285,319-327,494-497,512-514`
- 上游无 dirty-memory-ranges/state-only:全仓 grep 无命中(swagger `firecracker.yaml` 亦无)

### AgentEnv / FC fork

- pause 链:`AgentENV src/sandbox/firecracker/sandbox.rs:660-679,831-837`;state-only Diff:`instance.rs:457-464`
- 私有 API 语义:FC fork `v1.15.1-patch` `src/vmm/src/vstate/vm.rs:329-341`;内部 bitmap 只增不减:`memory.rs:825-827`(仅 dump_dirty 成功后 reset)
- 读也标脏:`devices/virtio/block/virtio/io/async_io.rs:89-117`(esp. 115);DAMON skip_anon:`AgentENV src/sandbox/firecracker/config.rs:30-46`;用户盘 Async:`sandbox.rs:1721-1727`
- mem 层物化:`overlaybd_snapshot.rs:717-784`;resume mmap:FC fork `memory.rs:646-672`;层链 32 上限:`overlaybd_snapshot.rs:37`
- FC 二进制:容器内 `deps/firecracker/1.15.1-patch-v1`(`--version` 实测 `v1.15.1-patch-v1`);pin:`config/deps_manifest.toml:1-3`;fork 源:github.com/kvcache-ai/firecracker 分支 `v1.15.1-patch`

### gVisor fork(`/home/keyang/AKernelWorkspace/gvisor`,HEAD `1985ab80b`)

- 双 MF 分叉:`pkg/sentry/kernel/kernel.go:913-923`;匿名 MF=memfd:`runsc/boot/loader.go:1091-1097`;tmpfs→k.mf:`pkg/sentry/fsimpl/tmpfs/tmpfs.go:227-257`
- 序列化与去零:`pkg/sentry/pgalloc/save_restore.go:593-691`;ExternalContent 仅私有 MF:`save_restore.go:341-382,189-196`;adopt 硬校验:`save_restore.go:1233-1259`;FICLONE:`save_restore.go:290-296`
- filestore 创建:`runsc/container/container.go:1246-1266`;私有 MF 装配:`runsc/boot/vfs.go:773-779,1225-1248`
- 压缩与 pages-file 互斥:`runsc/sandbox/sandbox.go:1827-1847`;AsyncPagesFileLoad:`save_restore.go:1432+`
- e2 负载(filestore 线):`AKernel_scheduler/prototype/cr-bench/run_e2.sh:63-71,184`

### CubeSandbox(`/root/CubeSandboxWorkSpace/CubeSandbox`)

- 三模式:`hypervisor/vm-migration/src/lib.rs:110-123`;分发:`hypervisor/vmm/src/memory_manager.rs:3111-3128`
- pagemap-anon:`hypervisor/vmm/src/pagemap_anon.rs:4-17`;soft-dirty:`hypervisor/vmm/src/soft_dirty.rs:8-51,267+`;降级:`memory_manager.rs:2501-2530`;clear_refs 成本:`memory_manager.rs:2475-2479`
- fast restore mmap:`memory_manager.rs:1326-1376,1509-1516`
- CubeCoW/reflink:`cubecow/README.md`;Cubelet 预 clone:`Cubelet/services/cubebox/appsnapshot.go:570-583`;clone 语义:`docs/blog/posts/2026-06-25-cubesandbox-snapshot-clone-rollback-deep-dive.md` §4
- PVM 内核仅无 KVM 路径:`docs/guide/pvm-deploy.md:24-38`;XFS reflink 要求:`docs/guide/quickstart.md:35-41`

### 实验产物(BareMetal)

- runsc 匿名线:`/root/cr-bench-bm/scripts/anon_scale.sh`;日志 `logs/anon_scale.out`(首轮,restore 失败=空工件边角)与 `logs/anon_scale2.out`(终版);沙箱日志 `logs/anon.log`
- AgentEnv:`/root/AgentEnvWorkSpace/aenv_anon_scale.sh` + `aenv_anon_scale2.out`;`shm512_verify.sh`(假阴性澄清);`redirty_test.sh`(读重脏因果实验:817/57/826ms)
