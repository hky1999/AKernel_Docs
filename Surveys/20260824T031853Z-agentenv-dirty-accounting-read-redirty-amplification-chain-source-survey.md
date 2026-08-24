# AgentEnv 脏账语义深挖:"DAMON 回收 → 重读 → async 标脏 → 整批重物化" 放大链 —— 源码逐环 Survey(含 Checkpoint/Resume 全流程与 overlaybd 交互)

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T031853Z`(同日扩充:新增 §2–§6,补齐 AgentEnv pause/resume 完整流程、overlaybd/ublk 交互、单 checkpoint 多实例语义)
> 缘起:解释 [023535Z 三线成本 survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §4.3 中的一句话——"脏账语义由 fork 自己的 bitmap 定义,于是有了*页缓存被 DAMON 回收 → md5sum 重读 → async 引擎标脏 → 下次 pause 整批重物化*"。本文把这条因果链的**每一环**拆到源码行级,并完整讲清 AgentEnv 的 checkpoint 流程、resume 流程、它与 overlaybd 的交互,以及"一份 checkpoint 恢复多个实例"的语义。
> 源码证据:
> - **FC fork**:`kvcache-ai/firecracker` 分支 `v1.15.1-patch`(AgentENV 容器内 `deps/firecracker/1.15.1-patch-v1`),本地 clone `/tmp/fc-fork`;
> - **AgentENV**:初版引 BareMetal `/root/AgentEnvWorkSpace/AgentENV`(commit `7f4a9b9`);**2026-08-24 已对照最新本地 clone** `/home/keyang/AgentInfraWorkspace/kvcache-ai/AgentENV`(commit `39bfa34`,比 `7f4a9b9` 新 12 个 commit)逐条复核——所有相关行号未变(仅 config.rs boot args 由 :30-46 移至 :30-53),**放大链三环在上游最新代码中均未被修复**(§10.1);
> - 因果实验数据:`/root/AgentEnvWorkSpace/redirty_test.sh`(2026-08-24,817/57/826 ms)。

---

## 1. 一句话版本与链路总览

AgentEnv 的 pause 成本 = O(**它自己定义的脏集**)。这个脏集不是"guest 写了什么",而是"**FC 内部位图上有没有位**"。位有两个来源:KVM dirty log(guest 真实写)和 **VMM 在块设备完成路径上无差别打的位(读也算)**。而 guest 内核的 DAMON 回收会周期性把文件页缓存逐出 → 任何重读文件的动作(md5sum、grep、重新编译)都会触发块设备读 → 读完成即打位 → 下一次 pause 把这些**内容根本没变**的页整批物化进 mem 层。

```
            guest 用户态                guest 内核                    host(VMM)
┌──────────────────┐   ┌──────────────────────────┐   ┌─────────────────────────────┐
│ md5sum /root/f.bin│──▶│ 文件页已被 DAMON 回收      │──▶│ virtio-blk 读请求            │
└──────────────────┘   │ (skip_anon=Y,只回收文件页)│   │ (用户盘 IoEngine=Async)      │
                       └──────────────────────────┘   └──────────┬──────────────────┘
                                                                 ▼ ①
                                              async_io.rs: mark_dirty_mem_and_unwrap
                                              (读完成,把 DMA 目标页写进 FC 内部位图)
                                                                 │
                                                                 ▼ ②
                                              GET /vm/dirty-memory-ranges (preserve 语义)
                                                                 │
                                                                 ▼ ③
                                              process_vm_readv 物化 → overlaybd mem 层
                                              (pause 时间 + 磁盘 = O(读集),非 O(写集))
```

三个标注点即三个"谁规定的":① FC fork 的 async IO 引擎;② FC fork 的私有 API 的 preserve 语义;③ AgentENV 的物化路径(§3、§5 展开)。

阅读路线:先看 §2 两个基础概念(什么是 `IoEngine=Async`、什么是 DAMON reclaim),再看 §3–§6 的全流程与 overlaybd 交互,最后 §7–§9 回到放大链的三环拆解。

---

## 2. 背景概念:两个名字是什么

### 2.1 "用户盘 IoEngine=Async" 是什么

Firecracker 的每块 virtio-blk 盘有一个 `io_engine` 配置,决定 VMM 进程**用什么方式替 guest 做磁盘 IO**:

| | `Sync`(默认) | `Async` |
|---|---|---|
| 实现 | 阻塞 `pread/pwrite` 系统调用(`sync_io.rs`) | **io_uring**(`async_io.rs` 的 `AsyncFileEngine`):请求作为 SQE 提交、完成从 CQQ poll |
| 读路径 | 直接读进 guest 内存 | 先读进**对齐 bounce buffer**,完成时再 `mem.write_slice` 拷进 guest 内存 |
| 附加能力 | — | 磁盘限速(rate limiter)依赖它 |
| 脏记账 | **无**(sync_io.rs 全文没有 `mark_dirty`) | **每个完成请求(含读)`mark_dirty`**(§7 环①) |

AgentENV 给 guest 配了三块盘(`sandbox.rs:1695–1740`):`/dev/vda` tools 盘 `IoEngine::Sync`;**`/dev/vdb` 用户盘(agent 的工作盘,rootfs 所在)`IoEngine::Async` + 预置 rate limiter**;`/dev/vdc+` 额外盘。所以 agent 负载的全部用户 IO 都走 async 引擎——这正是环①能生效的前提;若换 Sync 引擎,读重脏链路当场断掉(但也失去限速与读 DMA 记账)。

### 2.2 "guest 默认开 DAMON reclaim" 是什么、为什么开

DAMON(Data Access MONitor)是 Linux 内核的访存监测框架;**DAMON reclaim** 是建在其上的"主动回收"模块:内核自己监测哪些页冷,冷过阈值就主动把它们逐出内存,不等内存压力逼到被动回收。guest 跑的是 **AgentENV 自带的内核 `vmlinux-6.1.175`**(`config/deps_manifest.toml` pin,勿与宿主 6.8 混淆;DAMON reclaim 自 5.16 起主线可用),AgentENV 通过 **boot args 打开它**(`config.rs:30-53`,§9 环③全文引;注释要求与 `config/default.toml` 保持同步):

| 参数 | 值 | 含义 |
|---|---|---|
| `enabled` | Y | 打开主动回收 |
| `min_age` | 60 s | 页要**冷满 60 秒**才回收(保守) |
| `quota_ms/quota_sz` | 100 ms / 1 GiB 每 1 s | 回收吞吐上限 |
| `wmarks_high/mid/low` | 900/700/200 ‰ | 空闲内存水位:低于 70% 开始,高于 90% 停,20% 以下激进 |
| `skip_anon` | **Y** | **只回收 file-backed 页(页缓存),匿名页不动** |
| `wmarks_interval` | 5 s | 水位检查周期 |

**为什么开**:沙箱 guest 内存很小(模板 1–4 GiB),而 agent 负载读大量文件(repo、依赖、模型输出)。不开回收,页缓存会无限膨胀撑满 guest RAM——首停脏集、mem 层体积、resume 后 RSS 全部跟着涨。DAMON 把驻留集钉在"活跃工作集"上,是 AgentEnv 控制内存线成本的阀。

**代价**:它制造了"逐出 → 重读"循环——这正是放大链环③。一个旨在**缩小**脏集的机制,和环①②的记账语义组合后反而**放大**了脏集,这是本文的核心讽刺。

### 2.3 guest 页缓存的两条持久化线(与 DAMON 的关系)

guest 页缓存物理上就是 guest RAM 的一部分,对 host 没有任何特殊身份(FC 的 dirty tracking 是物理页级,分不清"页缓存"和"malloc 的堆")。于是它的持久化有**两条互相不知情的路径**:

| | 路径 A:写回(guest 主动) | 路径 B:内存 checkpoint(host 被动) |
|---|---|---|
| 机制 | guest kernel 把脏页缓存写回 virtio-blk → async 引擎 → overlaybd upper(rootfs 线) | pause 时凡在脏账里的 guest RAM 页(含页缓存页)物化进 mem 层 |
| 触发 | `fsync`/内核脏页阈值 | 每次 pause;判定 = 自上次物化以来被写过(EPT/async 打位) |
| 写回后状态 | guest 侧变**干净页缓存**(可零成本丢弃) | **写回不会把页移出脏账**——没有"已落盘"的跨边界通知 |

两线叠加 = file 变体实测 **≈2×M**(rootfs 封存 M + mem 层 M,同一字节各记一次)。反向对照:**未 fsync 的写**只走路径 B(upper 不增长),fork 的 child 仍能看到该文件——因为页缓存内容被 mem 层整体带走。

**DAMON 不持久化任何东西**,它的作用点在时间维度:页缓存驻留 → 重读命中内存,环①不触发;被 DAMON 回收(丢的是干净页,块设备上有权威副本)→ 重读退化为块 IO → 环①标脏 → **重新进入路径 B 的脏账**。即 DAMON 通过控制页缓存驻留时长,间接控制"哪些读会重新进内存 checkpoint 的账"。

**microVM 特有 vs gVisor**:gVisor 里应用文件读写直接落 host 内核管理的 filestore,页缓存天然持久化在宿主文件里,从不住在"可 checkpoint 的内存"里——不存在双记问题。microVM 因 guest 有独立内核,页缓存只能住 guest RAM,双记不可避免(CubeSandbox 的 anon∩soft-dirty 也只是记账层缓解)。

---

## 3. AgentEnv 完整 Checkpoint(pause)流程

编排层入口 `Orchestrator::pause_sandbox`(`orchestrator/service.rs:1062`)做状态机(CAS 到 Pausing)、钉住镜像引用、分配持久化目录,然后 `detach_sandbox_handle_and_route` 把沙箱从运行表中摘掉,调后端 `sandbox.pause(artifact_root)`(`sandbox.rs:279-305`)。真正的数据面在 `FirecrackerSandbox::pause_to_dir` → `snapshot_to_dir`(`sandbox.rs:660-681, 683-820`):

### 第 0 步:冻结 VM

`fc_instance.pause()` = `PATCH /vm {state: Paused}`——vCPU 停走,设备静默。此后 guest 不再产生任何内存写与 IO,是后面所有"读账本"的一致性点。

### 第 1 步:state-only 快照(只落 CPU/设备状态,~20 KB)

`create_state_only_snapshot`(`instance.rs:459-469`):`PUT /snapshot/create`,`SnapshotType::Diff`、**不传 `mem_file_path`**。FC fork 的 persist 侧是 `if let Some(mem_file_path) = ...` 才 dump 内存(fork `persist.rs:178`)——所以这一步只产出 `vm_state.bin`(vCPU/中断/设备状态,实测 ~20 KB)。**内存完全不经过 FC 的快照文件**,这是 fork 的第一处定制:把"内存搬运"从 FC 内部挪出来,交给外部的 overlaybd 管线(下一步)。

### 第 2 步:取脏账

`GET /vm/dirty-memory-ranges`(FC fork 私有 API,上游不存在)= `get_dirty_memory_ranges_preserve`(`vm.rs:329-344`):读 KVM dirty log(读即清)∪ FC 内部 AtomicBitmap,**并把 KVM 的位 OR 回内部位图**。返回结构含 4K 对齐的 range 列表(base_host_virt_addr/image_offset/长度)。语义细节(preserve、只增不减)在 §8 环②;"读到的到底是什么、怎么读出来的"在 §3.1 展开。

### 第 3 步:把脏页物化成 overlaybd mem 层

`snapshot_memory_to_overlaybd`(`sandbox.rs:823-848`)→ `convert_dirty_memory_to_overlaybd`(`overlaybd_snapshot.rs:759+`):

1. **range → 512B 扇区映射**:`dirty_ranges_to_segment_mappings` 把 4K 脏区切成 `OVERLAYBD_ALIGNMENT`(512B)扇区的 `SegmentMapping`(`offset`=内存镜像内目的扇区,`moffset`=FC 进程 HVA 源扇区,受 `Segment::MAX_LENGTH` 截断),校验对齐与区间不重叠;
2. **源不是文件,是 FC 进程本身**:`ProcessVmReader::new(firecracker_pid)` 封装 `process_vm_readv`——**跨进程直接读 FC 的虚拟内存**(AgentENV 是 FC 的父进程,有权限);
3. **压层**:`publish_memory_overlaybd_layer` 以 32 并发(`DIRECT_MEMORY_SNAPSHOT_COMPACTION_CONCURRENCY`)把映射集合写成 LSMT 格式的 `mem_overlaybd/overlaybd.commit`,先写 `.commit.tmp` 再 `rename` 封存。

### 第 4 步:组装 mem 层链

`build_mem_snapshot_image_config`(`snapshot_to_dir` 内,`sandbox.rs:695-720`):若本次是从快照 resume 来的(`LaunchMode::Resume`),把**继承的 lowers 链**与新的 commit 拼起来,产出 `mem_image.json`。层链超过上限(32,`overlaybd_snapshot.rs:37` + `560-588`)才触发 compact 合并。`split_runtime_suffix`(`overlaybd_snapshot.rs:104+`)保证"运行期本地层"必须排在已发布/缓存层之后,避免领养时悬空引用或层优先级错乱。

### 第 5 步:rootfs restack(封存可写层,同窗口零拷贝)

`restack_snapshot_overlaybd_rootfs`(`overlaybd_snapshot.rs:620-634` → `restack_snapshot_overlaybd_device :590-618`):让 ublk daemon 把当前 log-structured upper **close-seal 并 rename** 成 `rootfs/snapshot.commit`(同文件系统,零字节拷贝),再开一个全新空 upper。快照的 rootfs image config 被重写为"封存层作为只读 lower"(`sandbox.rs:738-800`)。

### 第 6 步:manifest 落盘

`FirecrackerSnapshotManifest` + `FirecrackerSnapshotConfig`(vm_state 路径、mem overlaybd 配置、rootfs 配置、两块盘的 virtual size、extra drives)写入快照目录。目录布局(`managed-snapshots/<sandbox_id>/<uuid-v7>/`):

```text
<snapshot_dir>/
  vm_state.bin                     # ~20KB,CPU/设备状态(FC fork state-only)
  mem_image.json                   # mem 层链配置(lowers 继承链 + 本次 commit)
  mem_overlaybd/overlaybd.commit   # 本次脏页物化层(512B 扇区 LSMT)
  rootfs/snapshot.commit           # 封存的旧 upper(只读)
  rootfs/image.json                # rootfs 层链配置
```

### 生命周期收尾:pause 之后 FC 进程死掉

编排层拿到 `FirecrackerPausedState`(`sandbox.rs:208-229`,只包着 snapshot_config)后,**运行句柄被丢弃 → FC 进程停止,guest RAM 释放**。pause 是"挂起-落盘-杀进程",不是挂起等待。这个细节决定了两件事:(a) 内存此时唯一权威副本在 mem 层里;(b) **下一次 resume 必然是新 FC 进程 → 内部脏位图天然清零**——这就是 023535Z §4.2 里 pause#2 平坦(52–58ms)的机制来源。

### 3.1 展开:第 2 步"取脏账"读到的是什么、怎么读出来的

**API 返回的是地址清单,不是内存内容**。FC fork 的返回结构(FC fork `memory.rs:34-52`):

```rust
pub struct DirtyMemoryRange {
    pub base_host_virt_addr: u64,  // FC 进程自己的宿主虚拟地址(给 process_vm_readv 用)
    pub image_offset: u64,         // 在全尺寸内存镜像内的偏移
    pub length: u64,
}
```

**不是完整内存状态,是增量脏集**:判定 = 自上次物化(或 FC 进程启动)以来被写过的 guest 物理页。fresh boot 后首停 ≈ guest 摸过的一切(大,但非全 RAM);resume 后的新进程从零记账,之后每次 pause 只拿 delta。**完整状态 = mem 层链并集**(restore mmap 全尺寸镜像,层链洞 = 零页)。

**匿名内存和 page cache 都包括,而且不止**——记账是 guest 物理页级、**身份盲**的(EPT 只知道"这页被写了"):

| guest 物理页归属 | 进不进脏集 |
|---|---|
| 用户匿名内存(堆/栈/MAP_ANON、tmpfs) | 写过就进 |
| **guest page cache** | 进(写页缓存=写内存;epoch 内新读入的页也算写→读重脏入口) |
| **guest kernel 自身**(slab/页表/内核栈/kernel data) | 进(全住在 guest RAM) |
| 设备 DMA 缓冲 | 进(async 打位兜底) |
| restore 后只读未写的页 | 不进(EPT dirty 只记写;MAP_PRIVATE 读不 CoW) |

身份盲正是 file 变体 2×M 双记的根源。**四层读取机制**:

1. **guest RAM 的宿主形态**:FC 为每个 memslot `mmap` 一段宿主匿名内存,`KVM_SET_USER_MEMORY_REGION` 注册;FC 自己持有 GPA→HVA 映射(dirty range 能给 HVA 的前提);
2. **KVM 硬件记账**:`track_dirty_pages=true`(AgentENV 默认,`config.rs:186`)开启 EPT 写保护 + **PML**:guest/DMA 首写某页 → EPT violation → gfn 记入 per-slot bitmap,运行时开销近零。`get_dirty_bitmap`(FC fork `vm.rs:309-327`)逐 slot 发 `KVM_GET_DIRTY_LOG`——**返回即清零**;无 bitmap 的 slot 走 `mincore` 近似(上游 fallback,AgentENV 场景不走);
3. **内部位图补账并合并**:vMMIO + async IO 完成的位(环①)与 KVM 位 OR 并集,连续 1 合并成 4K 对齐 range(image_offset 跨 slot 统一布局);
4. **内容读出**:AgentENV `ProcessVmReader`(`process_vm_reader.rs`)用 **`process_vm_readv(FC_pid, …)`** 让内核直接把 FC 地址空间里指定 HVA 段拷进自己的缓冲(一次 syscall 拷整段;AgentENV 是 FC 父进程,权限天然满足),再按 512B 扇区映射经 `compact_to` 写成 LSMT 层。

时序:全程 VM `Paused`(无新写)、FC 进程仍活着(快照完成才杀)——读到的是一致快照;传输量 O(脏页),与 guest RAM 总量无关。

### 3.2 fresh 实例的第一次 checkpoint:脏集的基线是"空"

两个账本(KVM dirty bitmap、FC 内部 AtomicBitmap)在 **VM 创建时都是全零**(dirty logging 随 machine-config 预置于 boot 前,`instance.rs:518-527`)。所以**不是从持久化状态 resume 的 fresh 实例,第一次 pause 的脏集 = 自上电以来被写过的所有 guest 物理页**——事实上的全量(对已写集而言),与 FC 上游"第一次 Diff ≈ Full"、CubeSandbox Incremental 首次写 base 是同一现象:

- **包含**:guest kernel 启动的全部写入(内核解压、initrd 解包、slab/页表/内核栈)、所有用户进程匿名内存、boot 以来读入/写过的 page cache(**含被 mmap 执行的二进制**,读入即写)、DMA 缓冲;
- **不包含**:从未被写过的 guest RAM——在 mem 层里是**洞**,restore mmap 读到洞 = 零页,恰好等于 fresh boot 零初始化 RAM 的语义,"没有基线"无需任何特殊处理。

**但 boot 脏集是每个模板只付一次的成本**(付在创建模板的那次 pause 里):从模板 spawn 的实例,基线 = 模板 mem 层,自己只记 spawn 后的写入——这就是稳态 pause 便宜(干净沙箱 52–143 ms,boot 脏集本身仅几十 MiB 量级)的原因。三种"第一次":

| 场景 | 第一次 checkpoint 的脏集 |
|---|---|
| fresh boot 实例(做模板) | 自上电全部写入(kernel 启动+OS+负载),≈ 事实全量 |
| 从模板 spawn 的实例 | 仅 spawn 之后的写入(基线 = 模板 mem 层) |
| fork 的 child | 仅 child resume 之后的写入(基线 = 父捕获的层) |

两个常被问到的精化:

- **"只 checkpoint 写集"与 stock FC 的对比要分对象**:对 **Full** 是本质区别(O(RAM)→O(写集),且 Full 还附带全内存 fault-in);对 **Diff** 是同量级、不同形态——stock Diff 也是物理 O(脏) 的稀疏文件,AgentEnv 的差异在 512B 粒度、内容寻址/可发布、**增量层链**(多次 pause 追加层而非反复覆盖一个稀疏文件)、restore 侧可多实例共享的 ublk 设备。
- **"resume 回来 1G"的说法要修正**:恢复的是**全尺寸虚拟镜像**(`mem_virtual_size` = 配置的 guest RAM),层链映射到整个地址空间,洞 = 零页;"1G"只是物理字节量,逻辑对象始终全量布局(与 CubeSandbox"增量记账、全量格式"同款选择)。
- **层链 append-only**:旧层封存后永不改写(mem commit tmp→rename;rootfs upper 封存为 `snapshot.commit` 后新开空 upper;compact 也是生成新文件而非原地改)——**只读不是约束而是共享的前提**:fork 的 child 与父能共享同一份 mem commit,靠的正是"谁都不会写它"。

成本汇总(1 GiB 脏,file 变体实测):pause ≈45ms + 0.65ms/MB;工件 = mem 层 O(脏) + rootfs 封存 O(upper 脏)(file 变体合计 ≈2×M,因为同一份数据两条线各记一次)。

---

## 4. Resume 流程:两种形态

### 4.1 原地 resume(同进程,`PATCH /vm {Resumed}`)

`FirecrackerSandbox::resume(&self)`(`sandbox.rs:851-858`)——最便宜的形态:同一个 FC 进程继续跑。因为 state-only 快照根本没动 guest 内存,原地 resume 后 VM 状态与 pause 前完全一致。用在两个地方:**fork 捕获窗口后放行源沙箱**(`sandbox.rs:340-362`),以及 pause/fork 捕获失败时的回滚(`sandbox.rs:296, 351`)。

**重要推论**:原地 resume **不清内部脏位图**(环②的清零点只有 `dump_dirty`,而 state-only 路径永远不调它)。所以 fork 之后源沙箱的下一次 pause 会把同一批脏页**再物化一遍**——fork 既是"复制"也是"读"(读取了账本),账本没清。

### 4.2 从快照重建(新进程,编排层 resume API 的实际路径)

`resume_sandbox`(`orchestrator/service.rs:1293+`)→ `LaunchPlan::Resume` → `build_from_paused_state` → `from_snapshot_config` → `start()`(核心在 `sandbox.rs:1500-1590`):

1. **新环境**:新 netns/tap0、spawn 全新 FC 进程、网络策略与 custom extension 钩子重挂;
2. **mem 层挂成共享 ublk 设备**:`UblkDeviceManager::get_or_create_shared_mem`(`ublk/device.rs:470+`)——**key 是规范化后的 `mem_image.json` 路径**:同 key 已有设备则升级 Weak 引用复用,释放中的 key 等通知。即**同一份快照恢复出的多个实例共享同一个只读 mem 块设备**(设备层 dedup);
3. **FC 加载**:`load_snapshot_file(vm_state, mem_device_path, network_overrides, ..., track_dirty_pages)` = `PUT /snapshot/load`,mem 后端为 File(指向 `/dev/ublkbN`)。FC 把该块设备 **`MAP_PRIVATE|MAP_FIXED` mmap 为 guest RAM**(fork `memory.rs:646-672`)——**restore 是惰性的**:只有被 guest 触碰的页才从 mem 层读入,首次写触发内核 CoW;
4. **收尾**:重设 MMDS、reconcile 磁盘限速(恢复的快照继承 pause 时的限速,需对齐当前节点配置)、`PATCH /vm {Resumed}` 放行。

实测 80–110 ms(首轮可达数百 ms,冷读 mem 层);`aenv start` 已发布快照 83 ms。

**两种形态的分工**:`fork`/回滚用原地(快,~ms);`pause→resume` 生命周期 API、跨启动恢复、从已发布快照新建,全部走重建(新进程,脏账清零)。这也是为什么稳态 pause/resume 循环里 pause#2 能平坦——不是账被清了,是**账本随着旧进程一起死了**。

---

## 5. 与 overlaybd/ublk 的交互全景

AgentEnv 最有辨识度的设计决定:**内存和磁盘用同一套分层块设备语义**。

### 5.0 先讲清楚:什么是 ublk 设备,"同一只读 mem 设备"共享了什么

**ublk 是 Linux 主线内核(6.0+)的 user block device 框架**:内核 ublk 驱动暴露一个真实的宿主机块设备节点 `/dev/ublkbN`,但设备数据不在内核实现——每个 IO 经 io_uring 转发给**用户态 daemon**,由 daemon 决定读写什么。AgentEnv 的 `uvm-ublk-daemon` 就是这样的后端:它在用户态打开 OverlayBD 的 LSMT 层文件,把"只读 lower 栈 + 可写 upper"聚合成逻辑盘挂成 `/dev/ublkbN`;FC 拿到的只是普通的 host 块设备路径(`add_drive` 挂给 guest → guest 看到 `/dev/vdb`),**完全不知道背后是分层文件**。

**guest rootfs 和 guest 内存确实都被 ublk 管起来了**,但形态不同:

| | rootfs 设备 | mem 设备 |
|---|---|---|
| 层栈 | 模板 lowers + **私有 log-structured upper(可写)** | pause 产出的 mem 层链(**只读**) |
| 设备粒度 | 每沙箱一个 | **按 mem_image.json 路径去重共享** |
| FC 怎么用 | virtio-blk 数据盘 | snapshot/load 的 File 后端:把 `/dev/ublkbN` **`MAP_PRIVATE` mmap 成 guest RAM** |

一个容易误解的 nuance:**正常运行期** guest RAM 只是 FC 进程里的普通匿名内存,不经 ublk;只有 **restore 时刻**才被映射到 mem 设备上。此后 guest 的每次写落在 `MAP_PRIVATE` 的私有 CoW 页里(不写穿,设备永远只读干净),这些脏页再由下一次 pause 经脏账物化成**自己的新层**。即"内存以 ublk 设备形态存在"是**静止态/恢复态**的表示,运行态是普通内存。

**"同一只读 ublk 设备"的确切含义**(`ublk/device.rs:470+` 的 `get_or_create_shared_mem`):key = 规范化后的 `mem_image.json` 路径。第一个恢复者创建设备;后来的恢复者发现 key 已有活引用(Weak 升级成功)**不建新设备,直接复用同一个 `/dev/ublkbX`**(日志 "reusing shared memory ublk device",同 dev_id;key 正在释放则等通知再试)。于是同一份快照的所有实例:

- **共享宿主侧资源**:一个 daemon 内 LSMT 实例、同一批 backing 文件的 page cache(后来者恢复时 mem 层页是热的——fork 快的原因之一)、一个设备号;
- **隔离在 FC 侧**:每个 FC 进程各自 `MAP_PRIVATE` mmap 同一设备,guest 写只弄脏自己进程的私有页,**物理页共享直到首写**(页粒度 CoW);
- 任何实例下次 pause 物化的脏层是**自己层链上的新 commit**,不动共享设备。

### 5.1 底座:ublk daemon + LSMT

单进程 `uvm-ublk-daemon` 集中管理所有 ublk 设备(`storage/ublk-daemon/src/main.rs:22-47`)。Rust 版 OverlayBD 在用户态把"只读 lower 栈 + 可写 upper"聚合成一个虚拟块设备,经 `/dev/ublkbN` 给 FC。写路径是 **log-structured upper**(`upper.data` + `upper.index`,默认 `UpperMode::LogStructured`):新数据永远追加,**索引粒度 512B 扇区 segment**(`lsmt/index.rs:8-16`)——不是 4K 页。读路径要查层栈(lower 命中率靠 premerged-index 缓存),实测写吞吐 213 MB/s。

### 5.2 三种"层"的交互

| 对象 | 层栈形态 | 谁共享 |
|---|---|---|
| rootfs | 模板 lowers(managed-layers,内容寻址,硬链接复用)+ 私有 log-structured upper | lowers 全实例共享 |
| **内存** | **mem 层链:每次 pause 追加一个 commit 层(= 本次脏页),继承链可 compact** | **同一 mem_image.json 的所有恢复实例共享一个只读 ublk 设备** |
| fork 派生 | 父 rootfs upper 封存为只读继承层 + 子私有空 upper;父 mem 层直接复用 | 同父所有 child 共享同一份 mem commit 与封存 upper |

交互的关键点:**FC 完全无感**。对 FC 来说 mem 就是一块 File 后端的块设备;ublk daemon 才知道那是 overlaybd 层栈;AgentENV 编排层知道层链语义(compact、继承、内容寻址发布)。三层解耦让"内存快照"免费获得了块设备世界的全部工具:内容寻址去重、P2P 分发、commit 发布(`snapshot create` = pause + 入库 + 广播,实测 6.97 s 全量一次,之后 83 ms 恢复)、512B 粒度增量。

### 5.4 改动边界:AgentEnv 动了谁的源码

| 组件 | 改了吗 | 说明 |
|---|---|---|
| Firecracker | **改了**(fork `v1.15.1-patch`) | 三处:state-only 快照(`mem_file_path` 可选)、私有 `GET /vm/dirty-memory-ranges`(preserve 语义)、async IO 完成路径 `mark_dirty` |
| **host KVM/内核源码** | **没改** | 只用主线接口:`KVM_GET_DIRTY_LOG`、KVM user memory region;ublk 驱动、`process_vm_readv` 均为主线能力(host 6.8 原样) |
| **guest Linux 源码** | **没改** | guest 跑原版内核;DAMON reclaim 是主线功能,只通过 **boot args**(`damon_reclaim.*`,§2.2)调参——配置,不是改码 |
| 用户态自研 | daemon/overlaybd/编排 | `uvm-ublk-daemon`、Rust 版 overlaybd、orchestrator,全在自己的 repo |

共性观察:CubeSandbox(soft-dirty 是主线配置项)、gVisor PR#14228 同样都不改内核——这类系统都把改动收敛在"自己的用户态组件 + 一个 VMM fork"里,这是可部署性的硬约束。

### 5.3 层链增长与 compact

每次 pause 追加一层;resume 继承整链;超过 32 层(`overlaybd_snapshot.rs:37`)在下次快照时 compact 成单层。长寿命沙箱的层链深度是隐形成本(每次读都要过栈)。

---

## 6. 一份 checkpoint 能否 resume 出多个实例?

**能,且有三条路径,共享语义各不相同:**

### 6.1 fork API(运行中父 → N 个子,共享最深)

`POST /sandboxes/{id}/fork {"count":N}` → `fork`(`sandbox.rs:340-393`,`orchestrator/service.rs:533+` 编排):

```
父 pause(捕获 snapshot_config)→ 父原地 resume(快,§4.1)
   → N 个 child = from_snapshot_config_with_override(snapshot_config.clone(), ...)
   → join_all 并行 start()(§4.2 重建路径)
```

子实例共享(实测 fork×3 @1GiB 脏:du 增量 937 MiB ≈ **一份** mem commit + 封存 upper;fork×2 工件树只有一个 `mem_overlaybd/overlaybd.commit`):

- **mem 层**:`get_or_create_shared_mem` 按 mem_image.json 路径 dedup → 同一**只读 ublk 设备**;FC 各自 `MAP_PRIVATE` mmap → guest 首写该页时**内核按页 CoW**,物理页在写之前全 child 共享;
- **rootfs 封存 upper**:作为只读继承层共享;每个 child 新开私有空 upper,块级 512B 隔离;
- **身份**:每个 child 有独立 sandbox_id / envd token / 网络。

实测 195–250 ms/child,子可见父全部内存+文件状态、子写不影响父(已验证)。**边界**:mem 层物化是 O(父脏) 一次性的(与 AKernel FICLONE 的零字节 fanout 有本质差距);且 fork 后父原地 resume,父的脏账未清(§4.1 推论),父下次 pause 会重复物化同一批页。

### 6.2 已发布快照新建(commit → N 个独立实例)

`snapshot create` 把 pause 工件按 digest 发布进 managed-layers(内容寻址 + P2P),之后 `aenv start <snapshot>` ×N 每次都走 §4.2 重建:各实例共享只读层(lower 栈 + 同一 mem 设备 key),各自私有 upper。83 ms/个。与 fork 的区别:源不要求活着;工件可跨节点分发。

### 6.3 paused 沙箱 resume(单实例续命)

`pause → resume` 生命周期对是**同一 sandbox_id 的挂起/续跑**,不是复制;实现上仍走重建(新 FC 进程),但对用户语义是"同一个 agent 继续"。

**一句话总结**:任何时刻,一份 checkpoint 的 `mem_image.json` 就是共享 key——fork、快照派生、resume 全部通过"同 key → 同一只读 ublk mem 设备 → MAP_PRIVATE 按页 CoW"实现多实例,磁盘成本只付第一次物化(§3 第 3 步),实例数增加不增加 mem 字节。

---

## 7. 环 ①:为什么"读也标脏"——async 引擎的 mark_dirty

### 7.1 源码(FC fork `src/vmm/src/devices/virtio/block/virtio/io/async_io.rs`)

每个用户盘请求完成(pop 出 io_uring 完成队列)时,无论读写,统一走 `mark_dirty_mem_and_unwrap`:

```rust
// async_io.rs:89-118
fn mark_dirty_mem_and_unwrap(
    self,
    mem: &GuestMemoryMmap,
    count: u32,
) -> Result<PendingRequest, RequestError<AsyncIoError>> {
    let WrappedRequest { addr, req, bounce_buf } = self;
    if let Some(addr) = addr {
        // If there is a bounce buffer, this was a read: copy data to guest memory.
        if let Some(ref bounce) = bounce_buf {
            let data = &bounce.as_slice()[..count as usize];
            if let Err(err) = mem.write_slice(data, addr) { ... }
        }
        mem.mark_dirty(addr, count as usize);      // ← :115,读写都打位
    }
    Ok(req)
}
```

```rust
// async_io.rs:364-377(pop 路径,唯一调用点)
pub fn pop(&mut self, mem: &GuestMemoryMmap) -> ... {
    ...
    let req = match wrapped_request.mark_dirty_mem_and_unwrap(mem, count) { ... }
```

而**同步引擎完全没有这个调用**:`sync_io.rs` 全文 grep `mark_dirty` 零命中(§2.1)。`mark_dirty` 落到 `GuestMemoryMmap`(FC fork `src/vmm/src/vstate/memory.rs:773-779`):

```rust
/// Mark memory range as dirty
fn mark_dirty(&self, addr: GuestAddress, len: usize) {
    // ignore invalid ranges using .flatten()
    for slice in self.get_slices(addr, len).flatten() {
        slice.bitmap().mark_dirty(0, slice.len());
    }
}
```

这个 `AtomicBitmap` 是 FC **内部**的脏位图——上游 Firecracker 也有它,但上游只用它捕捉 **vMMIO 等宿主侧对 guest 内存的写入**(`memory.rs:460-478`),并且 `dump_dirty`(diff 快照)成功后即 `reset`(取走即清)。FC fork 把它扩展成了"VMM 碰过的页"通用账本。

### 7.2 为什么 fork 要这么做(设计动机推断)

VMM 把数据**写进** guest 物理内存(`mem.write_slice`)这件事,guest 的 EPT dirty log **未必能记录**:KVM dirty log 通过 EPT 写保护 + PML 记录的是"经过 guest 页表视角的写";异步 io_uring 路径下,VMM 进程在自己的映射上直接写、且写发生在 `KVM_RUN` 之外,是纯宿主侧写。**若不打位,设备 DMA 灌入的数据会在下一次 diff 快照里丢失**——这是正确性必需的。

问题在于实现把它做成了**无差别**:对"VMM 写入 guest"(读请求的 bounce buffer 拷贝)打位是对的;但对**写请求**也打位(此时 VMM 根本没碰 guest 内存,数据是 guest 自己先写好的,EPT 已经记了),这一半是冗余;更关键的是对**读请求**打位意味着——

> **"这块页缓存重新进入 guest 内存"这一事件,被记账为"这块内存脏了"。**

对从未离开过 guest 内存的页,这个记账是保守但无害的;对**曾经被物化、后来内容从未改变**的页,它就是纯放大器。

---

## 8. 环 ②:preserve 语义——为什么脏位"只增不减"跨窗口累积

### 8.1 源码(FC fork `src/vmm/src/vstate/vm.rs:329-344`)

```rust
/// Returns dirty memory ranges without logically clearing dirty state.
///
/// Reading KVM's dirty log clears KVM's internal bits. Preserve those bits
/// in Firecracker's internal bitmap before returning the ranges so a later
/// snapshot still sees the pages if the external consumer fails.
pub fn get_dirty_memory_ranges_preserve(&self) -> Result<DirtyMemoryRanges, VmError> {
    let dirty_bitmap = self.get_dirty_bitmap()?;              // KVM log:取走即清
    let page_size = get_page_size().map_err(MemoryError::PageSize)?;
    self.guest_memory()
        .store_dirty_bitmap(&dirty_bitmap, page_size);        // OR 回内部位图(不丢)
    self.guest_memory()
        .dirty_memory_ranges(&dirty_bitmap, page_size)        // 返回 = KVM ∪ 内部
        .map_err(VmError::MemoryError)
}
```

`store_dirty_bitmap`(FC fork `memory.rs:860-880`)把 KVM 的位逐页 OR 进 `AtomicBitmap`;`dirty_memory_ranges`(`memory.rs:833-858`)返回**两张位图的并集**。内部位图唯一的清零点是 `dump_dirty` 成功后(上游语义,FC fork 保留在 diff 快照路径里)——但 AgentEnv 的 pause 走 **state-only 快照**(不传 `mem_file_path`,`persist.rs:178` 的 `if let Some(mem_file_path)` 不成立,**根本不调 `dump_dirty`**),所以内部位图在整个 FC 进程生命周期内**单调增长**。

### 8.2 这个语义是特性不是 bug(对"挂起-继续运行"场景)

`preserve` 的注释说明了意图:外部消费者(AgentENV)拿到 ranges 后**可能失败**(process_vm_readv 出错、磁盘满、上层放弃本次 pause)。如果取走即清,失败窗口内的脏页就永久丢了。preserve + 累积保证了"**至少一次**"物化:页一旦被打位,迟早会被某次成功的 pause 物化进 mem 层。

代价:脏账的语义从"自上次成功 checkpoint 以来写过的页"变成了"**自 FC 进程启动以来 VMM 记录过位的所有页**"——两式中环 ① 把"记录过位"扩到了读。账本的实际重置机制只有一个:**FC 进程死亡**(§3 收尾、§4.2)。

### 8.3 AgentENV 消费侧(物化)

`GET /vm/dirty-memory-ranges` → HVA → `process_vm_readv` 直接读 FC 进程内存 → 512B 扇区 SegmentMapping → overlaybd commit 层(`AgentENV src/sandbox/firecracker/overlaybd_snapshot.rs:717-784`,§3 第 3 步已逐步展开)。环 ①② 攒下的账在这里一次性兑现为 **pause 时间 + 磁盘字节**。

---

## 9. 环 ③:为什么重读一定会发生——DAMON 只回收文件页

### 9.1 源码(AgentENV `src/sandbox/firecracker/config.rs:30-53`,§2.2 已逐参数解释;最新 `39bfa34` 逐字未变)

```rust
/// Fallback boot arguments when `config.firecracker.boot_args` is not set.
/// ...
///   min_age        = 60 000 000 ns (60 s) — page must be cold for 60 s before reclaim
///   wmarks_high    = 900 (‰)      — stop reclaim when free pages > 90 %
///   wmarks_mid     = 700 (‰)      — start reclaim when free pages < 70 %
///   skip_anon      = Y             — only reclaim file-backed (pagecache) pages
const DEFAULT_BOOT_ARGS: &str = "\
    console=ttyS0 reboot=k panic=1 pci=off \
    damon_reclaim.enabled=Y \
    damon_reclaim.min_age=60000000 \
    damon_reclaim.quota_ms=100 \
    damon_reclaim.quota_sz=1073741824 \
    damon_reclaim.quota_reset_interval_ms=1000 \
    damon_reclaim.wmarks_high=900 \
    damon_reclaim.wmarks_mid=700 \
    damon_reclaim.wmarks_low=200 \
    damon_reclaim.skip_anon=Y \
    damon_reclaim.wmarks_interval=5000000";
```

要点:回收**一定会发生**(只要沙箱活得够久或水位够低);`skip_anon=Y` 把全部回收压力集中在**页缓存**上(guest 读过的每一个文件都是候选);回收一个干净页缓存页 = 丢弃内容,磁盘上有权威副本,**下一次任何人读它 = 重新发起块设备读**。guest 内核不知道 host 侧的块设备背后是 reflink/overlaybd,更不知道 host 侧有人会把"读"记账成"脏"。

### 9.2 三环咬合的完整时间线(file-1GiB 因果实验)

`/root/AgentEnvWorkSpace/redirty_test.sh`,单沙箱,`dd 1GiB urandom → /root/f.bin conv=fsync`:

| 步骤 | guest 侧发生 | host 侧账本 | pause 耗时 |
|---|---|---|---:|
| dd 完成 | f.bin 数据在页缓存 + 已写盘 | 写请求完成 → 环① 打位(1GiB) | — |
| **pause#1** | — | 物化 1GiB → mem 层;FC 进程死亡 | **817 ms** |
| resume(重建)+ 静置 | 无任何 exec | 新 FC 进程位图从零;页缓存若被 DAMON 回收,静默发生 | — |
| **pause#2**(无 exec) | — | 位图空 → 只物化 ~3 MB | **57 ms** |
| resume + `md5sum /root/f.bin` | 页缓存(可能)已被回收 → 每个块触发读 IO | **读完成 → 环① 再打位 1GiB** | — |
| **pause#3** | — | 再物化 1GiB → 新 mem 层 | **826 ms** |

注意 md5sum 是**纯读、零写入、guest 内存内容零变化**的操作——pause#3 的 826 ms 和 1GiB 磁盘增量全部花在搬运"和 pause#1 时一模一样的字节"上。放大倍数 = 重读率:agent 负载恢复后扫 repo、重读历史输出,放大率轻松到 100%。

### 9.3 什么条件下链路不咬合

| 条件 | 后果 | 证据 |
|---|---|---|
| 用户盘用 Sync 引擎 | pop 路径无 mark_dirty,读不记账 | sync_io.rs 零命中 |
| 不开 DAMON 回收 | 页缓存常驻,重读不触发块 IO | 参数关闭即可(但 mem 层会涨) |
| 重读的页还在页缓存里 | 不发块请求,环①不触发 | pause#2 平坦的原因 |
| 换全新 FC 进程后只写不读 | 只有真实写在记账 | shm 变体 pause#2 52–58 ms |
| gVisor(PR#14228)对照 | 匿名线只按**写**序列化,读不计脏 | 023535Z §5.2 |

**三环缺一不可**——这是它能同时通过 133500Z(shm-512M 仅首轮慢)与 041720Z(file-1G 三轮全慢)两批数据的检验、又被 redirty_test.sh 单变量因果实验钉死的原因。

---

## 10. 正确性与修复空间分析(如果 AgentEnv 要改)

这个放大链**不产生错误结果**——物化的页内容正确,只是浪费。修复按侵入度排序:

1. **读路径按需打位(治本,改 FC fork 一处)**:`mark_dirty_mem_and_unwrap` 只在 `bounce_buf.is_some()`(VMM 实际写入了 guest 内存)时 `mark_dirty`。写请求分支删掉是安全的吗?——guest 写盘前数据已在 guest 内存,EPT 已记脏;VMM 侧不写 guest。唯一要审计的是 vMMIO/间接描述符路径是否依赖此处兜底。预期效果:读重放完全不再入账,pause 回到 O(真实写集)。
2. **preserve 换成 ack 式清除(治本,协议改动)**:`dirty-memory-ranges` 返回后,由 AgentENV 在物化成功后显式调一个 `clear-dirty-ranges` 确认。保留"消费者失败不丢账"的意图,消掉单调累积。改动跨 FC fork + AgentENV 两处。顺带解决 §4.1 的推论(fork 后父重复物化)。
3. **DAMON 参数收紧(缓解,零代码)**:调高 `wmarks` 或加大 `min_age`,减少回收→重读频率。代价是 guest 驻留集变大、mem 层首轮更大——只是把成本从"重复物化"挪回"首次物化"。
4. **mem 层内容寻址去重(缓解,AgentENV 侧)**:物化时按页/段哈希,命中已有 managed-layer 则引用而非重写。读重放的页与上一轮内容相同,大概率全命中——把 826 ms 降为哈希计算时间(但 O(读集) 的扫描仍在)。

对读密集 agent 负载,1+2 组合后 AgentEnv 的 pause 语义才与 CubeSandbox soft-dirty(只记真实写)和 gVisor(只序列化写过的页)对齐。

### 10.1 上游现状核对:最新代码(2026-08-24,commit `39bfa34`)未修复放大链

对照本地最新 clone `/home/keyang/AgentInfraWorkspace/kvcache-ai/AgentENV`(HEAD `39bfa34`,2026-08-24;比本文实验所用的 BareMetal `7f4a9b9` 新 12 个 commit),**三环全部原样**:

| 环 | 最新代码状态 | 证据 |
|---|---|---|
| ① 读也标脏(FC fork) | **未变**——FC 依赖 pin 仍是 `1.15.1-patch-v1`(与 `7f4a9b9` 同一版本号、同一二进制),async `mark_dirty`/preserve 原封不动 | `config/deps_manifest.toml:1-3` |
| 用户盘 Async 引擎 | **未变** | `src/sandbox/firecracker/sandbox.rs:1727`(extra drives 亦 Async,:1769) |
| ③ DAMON boot args | **逐字未变**(仅文件行号 30-46 → 30-53) | `src/sandbox/firecracker/config.rs:30-53` |
| ② 的 AgentENV 消费侧 | **无 ack/清除/去重逻辑**(全仓 grep `clear_dirty`/`reset_dirty`/`ack`/内容寻址去重零命中;`get_dirty_memory_ranges` 调用方式未变) | `instance.rs:469-477` |
| 关键函数行号 | 全部未漂移:trait pause `:279`、fork `:340`、pause `:647`、pause_to_dir `:660`、snapshot_to_dir `:681`、resume `:851`、snapshot_memory_to_overlaybd `:823`、shared_mem `:1546`、load_snapshot_file `:1574`;instance `:457/:469`;service `:533/:1062/:1293`;overlaybd_snapshot `:589/:619/:756` | 逐条 grep 复核 |

`7f4a9b9..39bfa34` 的 12 个 commit 全部与本题无关:4 个 network 重构(egress CIDR/域名、host header 校验)、3 个 overlaybd/ublk io_uring 路径重构与预热门控、CLI 补全、clippy、安装脚本——没有触碰脏账、物化、层链语义。

**结论:截至 `39bfa34`,本文描述的读重脏放大链在上游最新代码中依然完整成立,§10 的四条修复建议均未被采纳。**(一个值得关注的平行动向:该 manifest 新增了 `firecracker.pvm v1.17.0-next.1` 依赖——PVM 路径与 CubeSandbox 的无-/dev/kvm 场景对应,值得后续跟踪其内存快照语义是否不同。)

---

## 11. 证据索引

### FC fork `v1.15.1-patch`(本地 `/tmp/fc-fork`)

- 读也标脏:`src/vmm/src/devices/virtio/block/virtio/io/async_io.rs:89-118`(esp. 115 `mem.mark_dirty(addr, count as usize)`),唯一调用点 `:374`(pop 路径);`sync_io.rs` 无 `mark_dirty`(grep 零命中)
- preserve 语义:`src/vmm/src/vstate/vm.rs:329-344`(`get_dirty_memory_ranges_preserve`,含"so a later snapshot still sees the pages if the external consumer fails"原注释)
- 内部位图实现:`src/vmm/src/vstate/memory.rs:773-779`(mark_dirty)、`:833-858`(dirty_memory_ranges = KVM ∪ 内部)、`:860-880`(store_dirty_bitmap,OR 回);清零点仅 `dump_dirty` 成功后(上游语义保留)
- state-only 快照不触发清零:`src/vmm/src/persist.rs:178`(`if let Some(mem_file_path)`)
- restore mmap(§4.2 第 3 步):`src/vmm/src/vstate/memory.rs:646-672`
- 上游对照(内部位图本源只记 vMMIO):上游 1.16.1 `memory.rs:460-478`,见 [023535Z survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §3.2

### AgentENV(双源核对:初版 BareMetal `7f4a9b9` + 最新本地 `/home/keyang/AgentInfraWorkspace/kvcache-ai/AgentENV` `39bfa34`,两者在本文所有引用点上行为一致,见 §10.1)

- FC fork 与 guest 内核 pin:`config/deps_manifest.toml`(firecracker.kvm `1.15.1-patch-v1`、kernel.kvm `vmlinux-6.1.175`、新增 firecracker.pvm `v1.17.0-next.1`)
- **盘配置与 IoEngine**:`src/sandbox/firecracker/sandbox.rs:1695-1740`(tools 盘 Sync / 用户盘 Async + rate limiter,预置于 boot 前;最新版用户盘 :1727、extra drives :1769)
- **DAMON boot args(含 `skip_anon=Y`)**:`src/sandbox/firecracker/config.rs:30-53`(§2.2/§9.1 全文引;`7f4a9b9` 时为 :30-46)
- **pause 编排**:`src/orchestrator/service.rs:1062-1200`(状态机、镜像引用钉住、detach 后 `sandbox.pause(artifact_root)`);后端 trait pause 与失败回滚:`src/sandbox/firecracker/sandbox.rs:279-305`
- **pause 数据面**:`sandbox.rs:647-658`(pause 入口+managed-snapshots 目录)、`:660-681`(pause_to_dir:冻结+落盘)、`:683-820`(snapshot_to_dir 全流程:mem 层、层链组装、rootfs restack、manifest、目录布局)
- **state-only 快照与脏账 API**:`src/sandbox/firecracker/instance.rs:443-477`(`create_state_only_snapshot`/`get_dirty_memory_ranges`,注释明言"memory data path is handled by AgentENV through dirty memory ranges")
- **mem 层物化**:`src/sandbox/firecracker/sandbox.rs:823-848`(snapshot_memory_to_overlaybd)、`overlaybd_snapshot.rs:759+`(`convert_dirty_memory_to_overlaybd`、`dirty_ranges_to_segment_mappings` 512B 扇区映射、`publish_memory_overlaybd_layer` tmp+rename 封存、32 并发)、`process_vm_reader.rs`(`process_vm_readv`)
- **rootfs restack**:`overlaybd_snapshot.rs:590-634`(capture_live → stage → snapshot.commit)、`sandbox.rs:738-800`(快照 rootfs image config 重写)
- **resume**:`sandbox.rs:851-858`(原地 resume)、`:1500-1590`(重建路径:netns、新 FC 进程、`get_or_create_shared_mem`、`load_snapshot_file`、限速 reconcile、resume);`orchestrator/service.rs:1293+`(resume 编排)、`:2143-2150`(`build_from_paused_state`)
- **共享 mem 设备**:`src/sandbox/ublk/device.rs:470+`(key=规范化 mem_image.json 路径,Weak 升级复用 + 释放等待)
- **fork**:`sandbox.rs:340-393`(pause→原地 resume→N child 并行 start)、`orchestrator/service.rs:533-600`(状态机与 children_spec)
- **层链/compact/发布**:`overlaybd_snapshot.rs:37`(32 层上限)、`:104+`(`split_runtime_suffix`)、`:560-588`;`src/snapshot/manager.rs:102-193`(commit 发布 + P2P)

### 实验(BareMetal,2026-08-24)

- 因果实验:`/root/AgentEnvWorkSpace/redirty_test.sh`(817/57/826 ms,file-1GiB)
- scaling 与假阴性澄清:`aenv_anon_scale.sh`、`shm512_verify.sh`(023535Z §4.2 表)
- fork 共享语义实测(1GiB,fork×2/×3):见 [163100Z survey](20260820T163100Z-agentenv-overlaybd-spawn-cr-cow-vs-akernel-survey.md) §3.4 与 [041720Z](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md) §4.2
