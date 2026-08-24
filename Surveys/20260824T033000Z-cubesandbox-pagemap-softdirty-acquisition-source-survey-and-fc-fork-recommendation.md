# CubeSandbox 宿主 pagemap / soft-dirty 脏页采集机制与完整 C/R 编排流程源码 Survey —— 兼"自维护 FC fork 该走哪条路"的推荐

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T033000Z`(同日扩充:新增 §2 与 AgentEnv 记账的本质区别、§6 完整 C/R 编排流程;全部代码引用迁移至本地最新 clone)
> 缘起:回答四个问题——(1) CubeSandbox 的宿主 `/proc/<pid>/pagemap` 和 soft-dirty 位到底是怎么拿到的;(2) 这套记账与 AgentEnv 的 KVM dirty log 记账有什么**本质区别**;(3) CubeSandbox 的完整 checkpoint/restore 编排流程是怎样的,reflink 到底管不管内存 image 和磁盘 image;(4) 如果我们自己维护一个 Firecracker fork,增量内存快照的记账应该学 AgentEnv 还是学 CubeSandbox。
> 源码证据(2026-08-24 已全部迁移到本地最新 clone):**`/home/keyang/AgentInfraWorkspace/tencentcloud/CubeSandbox`(HEAD `4efde86d`,2026-08-24 11:39 +0800)**,下文所有相对路径均以此为根。初版引用的 BareMetal `/root/CubeSandboxWorkSpace/CubeSandbox` 与 `/tmp/cube-src/` 副本已弃用。
> 姊妹篇:[AgentEnv 读重脏放大链 survey](20260824T031853Z-agentenv-dirty-accounting-read-redirty-amplification-chain-source-survey.md)(环①②③ 的对位分析)、[023535Z 三线成本 survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §6

---

## 1. 结论摘要

1. **先纠正一个常见前提:CubeSandbox 不是基于 Firecracker**。它是 **Cloud Hypervisor 的 fork**(`hypervisor/` 目录就是 CH 的 vmm/vm-migration crate 布局),与 FC 是两个独立代码库的 KVM microVM VMM。但二者 guest 内存模型相同(VMM 进程内 mmap + `KVM_SET_USER_MEMORY_REGION`),所以 §2 讨论的"谁都能读自己的 pagemap"在两个 VMM 上都成立——区别在于**账本放在哪、基线从哪来**(§2 展开)。
2. **CubeSandbox 没有碰任何 KVM 脏页接口,也不需要跨进程读别人**——它的 VMM 进程把 guest RAM 以 `MAP_PRIVATE` mmap 形式放在自己的地址空间里(restore 即 mmap 快照文件),所以"guest 写了哪些页"等价于"我自己的哪些页被 CoW 成匿名页/带上了 soft-dirty 位",答案躺在 `/proc/self/pagemap` 里,一次 `seek + read_exact` 批量拿走。
3. 两条互补的判定线:**pagemap-anon**(bit63 present + PFN → `/proc/kpageflags` 查 `KPF_ANON`,累计式)与 **soft-dirty**(`clear_refs(4)` 布防 + bit55,真窗口 delta);稳态用**二者交集**。
4. **完整编排是 Cubelet(Go)驱动、cube-runtime(CLI)转发、CH hypervisor(Rust)执行、cubecow(XFS reflink)供底座的四层流水线**;memory 和 rootfs 是**两个独立的 CowSnapshotObject**,但都由同一个 cubecow `FICLONE` 引擎管理——**reflink 同时服务内存 image(克隆上一代 memory 文件作覆写基座)和磁盘 image(rootfs volume→snapshot 提交)**(§6)。
5. 写入侧不产 raw delta:目标是**全尺寸 base 镜像的原地覆写**(base 由 FICLONE 预先 reflink-clone,覆写只破坏自己的 extent),消费方拿到的永远是自洽 full 镜像——**记账是增量的,格式是全量的**。
6. **推荐**:自维护 FC fork 走 **CubeSandbox 路线为骨架**(零 KVM API 改动、零私有 dirty API、FC File 后端本来就是 `MAP_PRIVATE` mmap、语义读中性、天然衔接 reflink 代际),**吸收 AgentEnv 的三点**:state-only 快照分解、KVM dirty log 作为大内存 VM 的加速/校验通道、preserve/ack 的教训(§8)。

---

## 2. 与 AgentEnv 的本质区别:不是"基于什么 VMM",而是"账本放在哪、基线从哪来"

### 2.1 先纠正前提

"二者都是基于 fc"——**前半句不成立**:CubeSandbox 的 `hypervisor/` 是 [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor) 的 fork(`hypervisor/vmm/src/memory_manager.rs`、`hypervisor/vm-migration/src/lib.rs` 均为 CH 的 crate 结构),不是 Firecracker。FC 与 CH 同为 KVM microVM VMM、guest 内存模型同构,但代码库完全独立(AgentEnv 改的 FC 内脏在 CH 里根本不存在对应物)。

### 2.2 为什么我此前只对 CubeSandbox 说"读自己的 pagemap 就能记账"

不是因为 FC 做不到,而是**两个原因叠加**:

**(a) AgentEnv 实际的实现没这么做**。它的账本是 FC fork 里的**软件位图**:`KVM_GET_DIRTY_LOG`(读即清)∪ FC 内部 `AtomicBitmap`(async IO 完成路径 `mark_dirty`,**读也打位**),经私有 API `GET /vm/dirty-memory-ranges` 暴露,再由 AgentENV 用 `process_vm_readv` 跨进程收割(姊妹篇环①②)。描述一个系统只能描述它实际的机制。

**(b) pagemap-anon 记账有一个 AgentEnv fresh-boot 场景不满足的前提:file-backed 基线**。`MAP_PRIVATE` mmap 下"匿名页 ⟺ 偏离 base 文件"这个推理,要求 guest RAM 确实 mmap 在**快照文件**上。这只在**从快照 restore 之后**成立:

| 场景 | guest RAM 的宿主形态 | pagemap-anon 过滤效果 |
|---|---|---|
| **从快照/模板 restore** | 快照文件的 `MAP_PRIVATE` mmap | **有效**:anon=CoW=真写集,只读页留在 base 的 page cache 里不重复存 |
| fresh boot(CH 或 FC) | `MAP_PRIVATE\|MAP_ANONYMOUS` 匿名内存(`memory_manager.rs:1535`) | **无效**:所有 present 页都是 anon,过滤一页都省不下来,还得付 kpageflags 遍历 |

CubeSandbox 的生产形态天生是"**一切沙箱从模板 restore 起步**"(AppSnapshot 建模板 → 后续创建全部走 restore),所以它的 VM 几乎总有 file-backed 基线,pagemap 记账总是有效;soft-dirty 线则不依赖基线(只看窗口内 bit55),但**同样只在"restore 起步"的稳态链里有意义**——首次快照没有前一个窗口,只能先写 full/anon 集(§5.1)。

**AgentEnv 原理上也能用这条路**:它的 restore 就是把 `/dev/ublkbN`(mem 层链)以 `MAP_PRIVATE|MAP_FIXED` mmap 为 guest RAM(姊妹篇 §4.2)——正是 §2.3 的前提。它没这么做纯粹是工程选择(这也正是本文 §8 推荐的自维护 fork 路线的立论基础)。

### 2.3 本质区别一张表

| | **AgentEnv 方式**(KVM dirty log + VMM 内部位图 + 私有 API) | **CubeSandbox 方式**(pagemap/soft-dirty 自省) |
|---|---|---|
| 账本放在哪 | **VMM 自己维护的软件位图**(KVM log OR 内部 AtomicBitmap) | **内核页表本身**(CoW→KPF_ANON;bit55 soft-dirty),VMM 只在快照时刻"读账" |
| 脏信息来源 | EPT/PML 硬件记账(guest 视角写)+ VMM 侧事件(**读完成也打位**,姊妹篇环①) | 页表状态,**读中性**:只有写才触发 CoW/anon |
| 基线要求 | 无——dirty logging 从 cold boot 就能开(位图全零起算) | **要求 file-backed 基线**(restore 起步);anon 线另需 `CAP_SYS_ADMIN` |
| 窗口语义 | preserve+单调累积(FC 进程生命周期内只增不减),清零点=进程死亡 | soft-dirty 真 delta(每轮 clear_refs 重布防);anon 累计(restore 起算) |
| 每轮固有开销 | PML 硬件记账近零 + O(dirty) 收割 | `clear_refs(4)` 全 PTE 遍历(每轮,multi-GiB 上数百 ms)+ kpageflags 随机读 |
| 外部采集 | `process_vm_readv` 跨进程读 FC 内存 | 无(VMM 自己 `pread` 自己的映射) |
| 降级路径 | 无(私有协议,一坏全坏) | soft-dirty → pagemap_anon → Full 三级(§6.2,探测/基线解析失败自动降级) |

一句话:**AgentEnv 让 VMM 自己记账(所以账本语义可以被 VMM 的实现细节污染——读重脏),CubeSandbox 让内核页表记账(语义由硬件 CoW 保证,VMM 只是读者)**。

---

## 3. 前置:为什么"读自己的 /proc/self/pagemap"就能给 guest 内存记账

Cloud Hypervisor(以及 Firecracker)的 guest 内存模型:VMM 在自己进程里为每个 memslot 建一段映射,再把该映射的 host 物理页经 `KVM_SET_USER_MEMORY_REGION` 塞给 KVM 当 guest RAM。**guest 的每一次内存写,物理上就是 CPU 对这段 VMM 进程映射的写**(经由 EPT 直写,不经 VMM 代码)。

CubeSandbox 的 restore 是 fast-restore 路径:把快照文件 fd 以 `MAP_PRIVATE` mmap 为 guest RAM(`hypervisor/vmm/src/memory_manager.rs:1326` `new_from_snapshot` → fast_restore check `:1383` → `open_read` 后作为 `snap_file` 传入 → `:1510-1548` 建区,`snap_file` 分支 `mmap_flags |= libc::MAP_PRIVATE`)。于是内核页表视角出现一个干净的二分:

| guest 对某页做过什么 | 内核页表状态 | pagemap 表现 |
|---|---|---|
| 从未触碰 | 无 PTE | present=0 |
| 只读(页错误读入,未写) | 映射到 base 文件的 page cache | present=1,KPF_ANON=0 |
| **写过** | **CoW 出私有匿名页** | present=1,**KPF_ANON=1**(且写后 bit55=1) |

**"guest 写集合"被内核免费记在了页表里**——这就是全部魔法的来源。VMM 不需要 ptrace 别人、不需要 KVM dirty log、不需要 fork 内核;它只是自己 mmap 了一块大内存,然后问内核"我这块内存现在长什么样"。`/proc/self/pagemap` 按 *8 字节/页* 索引(host 虚拟页号),`File::open` + `seek` + `read_exact` 一次批量读完整个区域(`pagemap_anon.rs:188-286`、`soft_dirty.rs:267-326`)。

权限细节:pagemap 的 bit63(present)/bit62(swap)/bit55(soft-dirty)对进程自身**始终可读**;但 **PFN(bits 0–54)与 `/proc/kpageflags` 只有 `CAP_SYS_ADMIN` 可读**(无权时 PFN=0)。所以 soft-dirty 线完全不需要特权,而 pagemap-anon 线(要拿 PFN 查 kpageflags)需要 VMM 以 root/保留该 capability 运行——代码里显式处理了无权情形(`pagemap_anon.rs:253-258` 返回 `NoCapSysAdmin` 错误)。

---

## 4. 线一:pagemap-anon(累计式"自 restore 以来写过的匿名页")

### 4.1 判定逻辑(`hypervisor/vmm/src/pagemap_anon.rs:188-286`)

```rust
pub fn get_anon_pages(host_addr: u64, length: u64) -> Result<Vec<bool>> {
    ...
    // 批量读 pagemap:start_page * 8 偏移,一次 read_exact
    for (i, item) in result.iter_mut().enumerate() {
        let entry = ...;  // 8 字节小端
        let present = (entry & PAGEMAP_PRESENT_BIT) != 0;   // bit63
        let swapped  = (entry & PAGEMAP_SWAPPED_BIT) != 0;  // bit62
        if swapped  { *item = true; continue; }   // 换出的匿名页 = guest 写过,必须存
        if !present { continue; }                 // 从未触碰,base 文件里有权威副本
        let pfn = entry & PAGEMAP_PFN_MASK;       // bits 0-54
        if pfn == 0 { return Err(NoCapSysAdmin); }
        // 逐 PFN 查 /proc/kpageflags(pfn * 8 偏移,seek + read 8 字节)
        if (flags & KPF_ANON) != 0 { *item = true; }  // bit12 = 匿名页(CoW 产物)
    }
}
```

三个判定分支的设计含义:

- **swapped → 脏**:匿名页才会被换出(文件页直接丢弃即可),且内核不保证换入后 soft-dirty 位保留,两条线都把 swap 页保守计脏(`soft_dirty.rs:313-317` 同);
- **!present → 干净**:没碰过的页在 base 文件里,零成本;
- **KPF_ANON → 脏**:CoW 是写的不可抵赖的证据,且**只可能由写触发**——读中性(这正是 AgentEnv async 引擎缺的性质,姊妹篇环①)。

粒度细节:全部下标/偏移用 `host_page_size()`(`sysconf(_SC_PAGESIZE)`,probe 一次缓存,`pagemap_anon.rs:36-47`)——注释明确是为了 **ARM64 64KiB 页**内核不至错位索引(配套单测用注入的 page_size 4K/64K 矩阵验证 `coalesce_pages_to_ranges :58-96` 的缩放)。

### 4.2 累计性(为什么需要第二条线)

anon 是"**自 restore 以来的并集**":页一旦 CoW,永远是匿名页,哪怕此后再没改过。它正确(内容确实可能偏离 base)、但不是 delta——长寿命沙箱每个快照都要重写整个累计集。CH 侧的 `SnapshotType::Incremental`(`hypervisor/vm-migration/src/lib.rs:110-142`,serde `"incremental"`)对应这条线。

## 5. 线二:soft-dirty(真窗口 delta)

### 5.1 布防与采集(`hypervisor/vmm/src/soft_dirty.rs:224-326`)

**先讲清楚 bit55 与"布防"到底是什么**。Linux 给进程的每个 PTE 留了软件自定义位 bit55(soft-dirty),内核规则:页被写过则置 1——但默认没人清它,置 1 后永远是 1,单独看只是累计标志。**"布防"(arm)= 向 `/proc/self/clear_refs` 写 "4"**:内核拿 mmap_lock 写锁**遍历本进程全部 VMA、清掉每一个 PTE 的 bit55**(全进程一个不落——"全局"即由此而来),同时给每个 VMA 重挂 `VM_SOFTDIRTY`——此后任何写都先触发 page fault,内核在 fault 处理里重装 PTE 并把 bit55 置回 1。清零不是目的,目的是**设陷阱**:布防之后,bit55=1 的语义变成"**从清零那一刻起这页被写过**",写动作自己留下证据。

一个稳态快照周期的时间线:

```text
t0   第 N 次快照:
     ├─ 读 /proc/self/pagemap,收走 present=1 且 bit55=1 的页(= 上一窗口以来写过的页,真 delta)
     ├─ 把这些页原地 pwrite 到 reflink 克隆的 base 文件上
     └─ 写 "4" 到 clear_refs —— 布防,bit55 全清零,新窗口开始
          │ guest 继续跑:
          │   写页 A → fault → bit55(A)=1        ← 留下痕迹
          │   只读页 C → 读 fault 装 PTE,bit55(C)=0 ← 读不留痕
          │   没碰的页 D → 无 PTE,present=0
t1   第 N+1 次快照:
     ├─ 再读 pagemap:bit55=1 的 = {A}(B 上窗口写过也没关系,账本 t0 清过)
     ├─ 只覆写这些页到新克隆的 base
     └─ 再布防,开下一个窗口
```

源码侧(`soft_dirty.rs:224-326`):

```rust
const CLEAR_REFS_SOFT_DIRTY: &[u8] = b"4\n";
const PAGEMAP_SOFT_DIRTY_BIT: u64 = 1 << 55;

pub fn clear_soft_dirty() -> Result<()> {
    // 写 "4" 到 /proc/self/clear_refs:全局清 bit55 + 重挂 VM_SOFTDIRTY(即布防)
}

pub fn get_soft_dirty_pages(host_addr: u64, length: u64) -> Result<Vec<bool>> {
    // 同样 seek + read_exact 批量读 pagemap:
    //   swapped → true(保守);present && bit55 → true(本窗口写过)
}
```

两条线的分工一句话:线一(pagemap-anon)是"**嫌疑集**"(CoW 过就永远匿名,累计只增,起算点=最近一次 restore),线二(soft-dirty)是"**现行集**"(窗口内真实发生的写,起算点=上次布防,每次快照后重置)。交集 = "内容可能变了 ∧ 这个窗口真的动了"。配套单测(`soft_dirty.rs:616-750`)明确断言了 delta 语义:round2 写页 2/3/4 → 只回这三页;round3 只写页 7 → 只回页 7,不回 2/3/4。**首次快照没有起算点**(布防前 bit55 是历史垃圾),所以先写 anon 全集、写完才做第一次布防——即 §7.1 的"惰性布防";布防后头一个窗口里部分非写的 PTE 装载也会置位(线二单独用的污染源之一),这是必须取交集的又一原因。

### 5.2 能力探测的 round-trip 技巧(`soft_dirty.rs:156-215`)

写 "4" 到 clear_refs 在 `CONFIG_MEM_SOFT_DIRTY=n` 的内核上**也会成功(静默 no-op)**,返回值不可信。探测改用真实往返:mmap 一页匿名页 → 写一字节(首写装 PTE,若内核支持则 VMA 的 VM_SOFTDIRTY 传播到 PTE bit55)→ 读 pagemap 看 bit55 是否为 1。**只有确认支持才做那次昂贵的全局布防**,不支持时连 mmap_lock 写锁的 PTE 全量遍历都省掉,调用方静默降级。

### 5.3 为什么要交集而不是单用 soft-dirty(`soft_dirty.rs:422-490`)

源码注释把三种近似的偏差说得很清楚:

- **anon-only**:正确但累计(§4.2);
- **soft-dirty-only**:两个污染源——(a) host 侧落在 file-backed page cache 上的写(内容与 base 磁盘副本相同,重存纯浪费);(b) 刚布防完的第一个窗口里,任何装 PTE 的写(包括 fault 进非匿名页的)都会置位;
- **anon ∩ soft-dirty**:恰好 = "内容可能偏离 base ∧ 本窗口真的改过",既最小又安全。

---

## 6. 完整 C/R 编排流程(Cubelet → cube-runtime → CH → cubecow 四层)

### 6.0 组件分工与硬前提

| 层 | 代码 | 职责 |
|---|---|---|
| **Cubelet cubebox 服务**(Go) | `Cubelet/services/cubebox/*.go` | 编排:创建/暂停沙箱、准备 CoW 工件、拼 cube-runtime 命令行、维护本地 snapshot catalog |
| **cube-runtime**(CLI) | 经 `buildCubeRuntimeSnapshotArgs`(`appsnapshot.go:604`)拼参:`snapshot --app-snapshot --vm-id <id> --path <path> --force --snapshot-type <type> [--resource --disk --pmem --kernel --container-id] [--memory-vol <url>]` | 转发层:让 CH hypervisor 执行快照 |
| **CH hypervisor fork**(Rust) | `hypervisor/vmm/src/memory_manager.rs` | 执行:三种 snapshot type 的内存采集(§4/§5)+ 状态落盘 |
| **cubecow**(Rust) | `cubecow/src/engine/reflink.rs` | 存储引擎:XFS reflink 卷/快照对象,**唯一后端就是 FICLONE**(`const FICLONE: libc::c_ulong = 0x40049409`,reflink.rs:87) |

**硬前提**:`storage_backend=cubecow`——AppSnapshot/CommitSandbox 入口都检查(`appsnapshot.go:85-89`、`template_ops.go:74-78`),不是 cow 后端直接 PreConditionFailed。所有工件(memory 与 rootfs)都是 reflink-capable 文件系统(xfs `-m reflink=1`)上的普通文件。

**memory 与 rootfs 是两个独立的 `CowSnapshotObject`**(`template_ops.go` 中 `memoryObject, rootfsObject` 并行声明、并行 cleanup/deactivate),snapshot 目录布局:

```text
<snapshot_dir>/cubebox/<templateID>/<CPU>C<MEM>M/
  sandbox_spec.json      # resource/disk/pmem/kernel spec(重建用)
  memory.dev             # cubecow memory 卷的 dev path(一行文本,appsnapshot.go:776-780)
  ...                    # CH 快照工件(vm state 等,cube-runtime 落盘)
# 底层(cubecow 引擎目录,reflink.rs:20-28 注释的布局):
#   vol-A/<vol-A>      ← 卷主文件(FICLONE 源)
#   vol-A/<snap-1>     ← FICLONE(<vol-A>)
#   vol-A/<snap-2>     ← FICLONE(<snap-1>) —— 链式扁平化
```

### 6.1 AppSnapshot:从零建模板(永远 full)

`AppSnapshot`(`appsnapshot.go:57-455`)的七步:

1. **Create 临时沙箱**(fresh boot,templateID_0;PreConditionFailed 时 destroy 重试一次);
2. 取 snapshot spec(resource/disk/pmem/kernel);
3. **`storage.CreateMemoryVolume(templateID, memorySizeBytes)`**(`Cubelet/storage/cow_store_ops.go:85`)= cubecow 建一个**空 memory 卷**;
4. **`executeCubeRuntimeSnapshot(..., snapshotTypeFull)`**(`appsnapshot.go:636`):**这里必然 full**——源码注释明说:"AppSnapshot builds a brand-new template from a fresh sandbox: there is no base memory blob to overlay onto… Incremental is reserved for CommitSandbox"(`appsnapshot.go:285-289`)。这也印证 §2.2:fresh boot 没有 file-backed 基线,增量线无从谈起;
5. **`storage.CommitRootfsFromBuild(templateID)`**(`cow_store_ops.go:72`)= 把 build 阶段的 rootfs 卷经 `CommitTemplateRootfs` **reflink 提交**成模板 rootfs 快照;
6. **Destroy 临时沙箱** → `deactivateCowSnapshotObjects`(memory+rootfs 两个对象);
7. `rename(tmp→final)` + 写 snapshot catalog(`SnapshotCatalogEntry`:MemoryVol/RootfsVol/Kind/SizeBytes/ComponentVersions…)——后续 create-from-template 与清理全靠它解析物理引用。

注意第 4 步之前还有 `collectEnvdVersion`(containerd Exec)——必须在 cube-runtime 把 guest 标记为 app-snapshotting(禁 exec)**之前**执行。

### 6.2 CommitSandbox:运行中沙箱提交为模板(三级降级,reflink 主场)

`CommitSandbox`(`Cubelet/services/cubebox/template_ops.go:30-230+`)。与 AppSnapshot 的本质区别:**沙箱正在运行、且(通常)是从某个快照 restore 起步的**——于是有基线可增量。

流程:validate(lifecycle 锁、目标校验)→ 取 spec → **`storage.CommitRootfs(sourceRootfs, templateID)`**(`cow_store_ops.go:56`,live rootfs 卷 → `CommitTemplateRootfs` reflink 快照)→ **`prepareCommitMemoryArtifact`**(`Cubelet/services/cubebox/snapshot_base_memory.go:148-243`,三级降级选 memory 工件 + snapshot type):

| Tier | 条件 | memory 工件 | snapshot type | 语义 |
|---|---|---|---|---|
| **1(happy path)** | 运行绑定快照的 memory 文件可解析 | **FICLONE 克隆上一代 commit 的 memory**(`storage.CommitMemoryFromBase`,`cow_store_ops.go:98`) | `soft-dirty` | 克隆文件已含上一代全部字节,满足 soft-dirty "destination must hold every still-clean page" 前提;hypervisor 只覆写本窗口真 delta,磁盘写放大最小 |
| **2** | 绑定快照被删(如 operator 删了最近一次 commit) | FICLONE 克隆**最近一次 restore** 的 base | `incremental`(pagemap_anon) | anon 集自 last restore 起算,恰好与 last-restore 文件配对 |
| **3** | 两个 base 都没了(如源模板被删而 VM 还在跑) | `CreateMemoryVolume` 空卷 | `full` | 正确但最贵 |

**Tier 1/2 必须区分 snapshot type 的原因**(源码注释,snapshot_base_memory.go:211-218):soft-dirty 位图在**上一轮 commit** 时清零,所以它与"上一代 commit 的文件"配对;把它配给更老的 last-restore 文件,窗口和文件就错位了。pagemap_anon 的 anon 集在**最近一次 restore** 时重置,所以它与 last-restore 文件配对。**base 与账本必须同步重置**——这是整套系统最精妙的约束,也是 AgentEnv 的 preserve 单调累积(姊妹篇环②)反面教材的正面解法。

基线解析(`snapshot_base_memory.go`):
- **Tier 1 源** = `resolveBaseSnapshotID`(:41):rollback label > create 时 runtime-snapshot annotation > 原始 template annotation;
- **Tier 2 源** = `resolveRestoreBaseSnapshotID`(:86):restore label(**Commit 不推进它,只有 Create 的 restore 路径和 Rollback 写**;被 opaque 恢复源污染时置 invalid,`snapshot_runtime_binding.go:91`);
- 两者共用 `resolveMemoryObjectFromSnapshotID`(:121):catalog 查 MemoryVol/Kind → cubecow 解析 dev path;任一环节断裂返回哨兵 `ErrNoBaseMemoryForIncremental`(:19)→ 上层**优雅降级而非硬失败**。

**成功路径的关键闭环**(`template_ops.go:186-196` 注释):cube-runtime 返回成功 = hypervisor 已把 delta 写进 memory 文件**且(soft-dirty 路径)已执行 `clear_soft_dirty()` 开启下一窗口**。Cubelet 立即 `setRuntimeSnapshotBindingLabels(cb, templateID, now)` 把下一轮 commit 的 base 钉到本次——后续失败导致工件被删时,过期绑定会把下一轮 commit 自然路由到降级分支,自洽安全。

### 6.3 Pause(pause_cow.go):挂起 = full 快照 + 摧毁活沙箱

`updateWithPauseCow`(`Cubelet/services/cubebox/pause_cow.go:73+`)产出与 CommitSandbox 同布局的目录,但 memory 走 `CreateMemoryVolume` 空卷 + **full**——pause 的恢复源对 Cubelet 是 **opaque**(CubeShim 内部 full 快照,Cubelet 无法 reflink 它,`snapshot_runtime_binding.go:81`),且该恢复路径会把 restore label 置 invalid,让下一次 CommitSandbox 的 Tier 2 正确跳过。rootfs 的 `CommitRootfs` 特意排在 `PauseToSnapshot` **之后**(注释:"disk snapshot matches the frozen memory")。pause 完成后同样写 memory.dev/sandbox_spec.json 并完全 destroy 活沙箱——挂起即落盘杀进程,与 AgentEnv 的 pause 收尾同型。

### 6.4 Restore / 从模板创建:fast-restore MAP_PRIVATE,闭环回 §3

创建/恢复路径读取 snapshot 目录(memory.dev → cubecow dev path、sandbox_spec.json → 重建参数),起一个全新 CH 进程,内存侧进入 `MemoryManager::new_from_snapshot`(`hypervisor/vmm/src/memory_manager.rs:1326`):

1. `support_fast_restore_check(config)`(:1383):非 shared 内存、无 virtio-mem 热插拔 → fast;
2. fast 路径把 memory 文件 `open_read` 作为 `snap_file` 传入建区 → `:1510-1548` 中 `snap_file` 分支 **`mmap_flags |= libc::MAP_PRIVATE`**(PROT_READ|PROT_WRITE, MAP_NORESERVE|MAP_PRIVATE)——guest RAM 从此 mmap 在快照文件上,§3 的页表二分前提成立,**CoW 循环闭环**;
3. 非 fast(shared/virtio-mem)回退 `fill_saved_regions` 逐段读填(慢路径);
4. restore 成功后 Create/Rollback 路径 stamp `MasterAnnotationRuntimeRestoreSnapshotID`——Tier 2 的账本锚点。

**多实例共享**:同一模板的 memory 文件/rootfs 快照可被任意多个 restore 并发 `MAP_PRIVATE` mmap——页粒度 CoW,物理页共享直到各实例首写;rootfs 侧每个实例从模板 snapshot 再 FICLONE 出自己的私有卷(cubecow 的 volume→snapshot→volume 派生)。与 AgentEnv "同一只读 mem 设备 + MAP_PRIVATE" 的共享语义同构,只是底座从 ublk/OverlayBD 换成了 XFS reflink。

### 6.5 reflink 到底管什么(对用户问题的直接回答)

**内存 image 和磁盘 image 都管,且都在 Cubelet/cubecow 层发生,hypervisor 完全不知情**:

| image | reflink 用法 | 时序 |
|---|---|---|
| **memory** | Tier 1/2:`CommitMemoryFromBase` = FICLONE 上一代(last-commit / last-restore)memory 文件 → 作 hypervisor 覆写基座;覆写只破坏新快照自己的 extent,**上一代完好** | cube-runtime 执行**之前**(Cubelet 先备好工件再调 CLI) |
| **rootfs** | `CommitRootfs`/`CommitRootfsFromBuild` = `CommitTemplateRootfs`(卷→快照,FICLONE);cubecow 引擎目录里 snap 链式扁平化(vol-A/snap-2 = FICLONE(snap-1)) | 同上 |

hypervisor 侧看到的只是一个可写 dev path,它只管 pwrite delta 页;"增量写放大被 reflink 吸收"整个发生在文件系统 extent 层——与 gVisor PR#14228 的 clone-on-adopt 是同一模式(023535Z §5)。cubecow 还有 S3 后端(`cubecow/src/engine/s3.rs`)做快照跨节点 export,`export_uuid`/`export_status` 贯穿 `Volume` 信息结构(`cubecow/src/lib.rs:29` 起)。

---

## 7. 写入侧与成本(内存采集的落盘细节)

### 7.1 首快照的惰性布防(`memory_manager.rs:2496` `send_soft_dirty_memory`)

不在 boot/restore 时布防(`clear_refs` 是全 PTE 写锁遍历,multi-GiB guest 上数百 ms,是 pause 尖峰的主导项),而是**第一次快照时**:写一份 anon 全集 → `probe_soft_dirty_support()`(成功即含布防)→ 置 `soft_dirty_armed`(`memory_manager.rs:1297` 初始化为 false)。稳态路径:交集过滤 → 原地覆写 → **每轮末尾再 `clear_soft_dirty()` 重布防**(这次布防是每窗口重复成本)。任何失败(内核不支持、clear 失败)→ 降级 anon 路径,`armed=false`,下轮重试;覆写失败则 disarm——**布防只由写盘成功驱动**,即 §8 要吸收的 ack 语义。

### 7.2 原地覆写:增量记账、全量格式

两条增量线都要求目标 base 文件存在(`memory_manager.rs:2388` `send_pagemap_anon_memory` 显式报错让调用方先 Full;soft-dirty 同),然后:

```
gpa_to_file_offset 表(线性布局)→ 对每个 filtered range:
    save_range_to_file(memory_file, range, file_off)   // pread guest 内存 → pwrite 到偏移
```

产物是**全尺寸、自洽的 full 镜像**——VMM 其余部分(restore、迁移接收端)完全无感知,不需要"base+delta 链"的恢复协议。这正是 §6.2 三级降级能随便切换 snapshot type 的原因:所有 type 的产物格式相同。

### 7.3 每次快照的成本清单

| 项 | 复杂度 | 备注 |
|---|---|---|
| pagemap 批量读 | O(区域页数) 顺序读(8B/页,≈2 ms/GiB) | 每轮 ×2(anon + softdirty 各一次读) |
| kpageflags 逐 PFN 查 | O(present 页数) **随机 seek** | anon 线独有;最值得优化的点(可按 PFN 升序排序化随机为顺序) |
| `clear_refs(4)` 布防 | O(全部 PTE),mmap_lock 写锁 | **每窗口重复**;multi-GiB 上数百 ms,pause 主导项(源码注释自认,`memory_manager.rs:2475-2479`) |
| 布防后首个写页 | 每页一次 fault | 与 KVM dirty log 的 EPT 写保护同型,摊在运行期 |
| 覆写 | O(交集脏页) 顺序写 | reflink 保护上一代 |

---

## 8. 推荐:自维护 FC fork 学哪条路

### 8.1 结论:CubeSandbox 为骨架,AgentEnv 为补丁

**推荐以 CubeSandbox 方式为主**,四个理由:

1. **前提免费**:FC 的 File 后端 restore 就是 `MAP_PRIVATE` mmap(上游 1.16.1 `persist.rs:516-526` → `memory.rs:352-362`),CubeSandbox 的整个模型(写=CoW=anon=bit55)在 FC 上原样成立,一行内存管线都不用改。而 AgentEnv 方式要动 KVM 交互、bitmap 语义、IO 引擎三处 FC 内脏,每处都是长期维护负担(async/sync 不一致就是前车之鉴)。
2. **语义正确**:读中性 + 真 delta + 窗口重置,直接避免姊妹篇整条"DAMON 回收→重读→标脏→重物化"放大链的账目污染(guest 内核把换出的页缓存重读进 guest RAM 的那次写仍会记——但那是真实写;被消灭的是"重放读的 DMA 目标页被无差别入账"这类虚增)。
3. **生态咬合**:原地覆写 + FICLONE 基座 = 与我们 PR#14228 的"生命周期绑定、内容不进 image"哲学同构;跨节点搬 base+delta、代际共享、多实例 MAP_PRIVATE 共享全部顺下来,不需要 OverlayBD 层链那套 compact/32 层上限协议。Cubelet 的"base 与账本同步重置"(§6.2 Tier1/2 约束)是现成的正确性蓝图。
4. **可降级、可测试**:soft-dirty→anon→Full 三级探测降级 + 全套页级单测(round-trip delta 断言)是可以带着走的工程资产。

**同时吸收 AgentEnv 的三点**(都是低成本高收益):

- **state-only 快照分解**(`vm_state.bin` 与内存 artifact 分离,~20KB 控制态):fork 改动极小,与任何内存方案正交;
- **KVM dirty log 作为大内存可选通道**:multi-GiB guest 上 `clear_refs` 每轮数百 ms 的 PTE 遍历会重新变成 pause 主导项,而 PML 硬件记账近零——可做"小内存 pagemap / 大内存 dirty log"双模,或用 KVM log 做交集校验(两个独立来源的交集是极强的正确性防线);它还覆盖 pagemap 路线覆盖不了的 **fresh-boot 窗口**(§2.2);
- **preserve/ack 的教训**(而非其实现):外部消费者可能失败,账目清零必须由消费成功的 ack 驱动——CubeSandbox 的"布防在写盘成功之后"已经是这个语义,照抄即可。

**两个必须提前处理的坑**:

1. **`CAP_SYS_ADMIN`**:FC production 用 jailer 降权运行,pagemap PFN/kpageflags 会变 0/不可读 → anon 线失效。要么保留该 capability,要么实现纯 soft-dirty 模式(bit55/present 自身可读,无需特权;代价是失去 anon 交集,需按 CubeSandbox 注释处理 file-backed 写污染),要么 fallback 到 Full。
2. **布防时机**:抄"首次快照惰性布防 + 每轮末尾重布防",并把 `clear_refs` 耗时打进 pause 分相日志;若 pause 尖峰不可接受,退化为 anon 累计 + 周期性 Full 重置(即 CubeSandbox `Incremental` 模式的姿态)。

---

## 9. 证据索引(全部为本地 `/home/keyang/AgentInfraWorkspace/tencentcloud/CubeSandbox`(HEAD `4efde86d`,2026-08-24)相对路径,行号逐一复核)

### hypervisor(Cloud Hypervisor fork)

- pagemap-anon 判定:`hypervisor/vmm/src/pagemap_anon.rs:188-286`(`get_anon_pages`:bit63/62、PFN mask `(1<<55)-1`、KPF_ANON=bit12、swapped→true、NoCapSysAdmin :253-258);`filter_memory_ranges_by_pagemap_anon :301-362`;host page size:`:36-47`(ARM64 64KiB 防坑);合并:`coalesce_pages_to_ranges :58-96`(4K/64K 矩阵单测 `:429-450`)
- soft-dirty:`hypervisor/vmm/src/soft_dirty.rs`(`clear_soft_dirty :224-249`(clear_refs "4" + 耗时日志)、`get_soft_dirty_pages :267-326`(bit55 采集、swap 保守 :313-317)、`probe_soft_dirty_support :156-215`(round-trip 探测)、`filter_memory_ranges_by_anon_and_soft_dirty :422-490`(交集 rationale 与实现));delta 语义单测:`:616-750`;交集单测(含 CAP_SYS_ADMIN skip):`:765` 起
- 内存采集编排:`hypervisor/vmm/src/memory_manager.rs:2388`(`send_pagemap_anon_memory`:base 必须存在 + 原地覆写 + gpa→file offset)、`:2496`(`send_soft_dirty_memory`:惰性布防、交集、覆写后重布防、失败 disarm 降级)、`:2475-2479`(布防成本注释)、`soft_dirty_armed` 初始化 `:1297`
- fast restore:`memory_manager.rs:1326`(`new_from_snapshot`)、`:1383`(`support_fast_restore_check`)、`:1510-1548`(建区 mmap flag 选择——`snap_file` 分支 `MAP_PRIVATE` `:1515`、fresh-boot `MAP_PRIVATE|MAP_ANONYMOUS` `:1535`);慢路径 `fill_saved_regions :1370-1375`
- snapshot type 枚举:`hypervisor/vm-migration/src/lib.rs:110-142`(`Full`/`Incremental`/`SoftDirty`,serde `"full"`/`"incremental"`/`"soft-dirty"`)

### Cubelet(Go)

- **AppSnapshot(模板首建,永远 full)**:`Cubelet/services/cubebox/appsnapshot.go:57-455`(七步编排、失败 forceDestroy 回滚);`storage.IsCowBackend` 前提 `:85-89`;"Incremental is reserved for CommitSandbox" 注释 `:285-289`;`writeMemoryDevFile :776-780`;catalog 写入 `:407-435`
- **snapshot type 常量与注释**:`appsnapshot.go:561-586`(full/incremental/soft-dirty 三常量,soft-dirty 的 "destination MUST already contain a valid base… Cubelet guarantees this precondition by reflink-cloning the binding base" 注释 :570-577)
- **cube-runtime CLI**:`buildCubeRuntimeSnapshotArgs :604`、`executeCubeRuntimeSnapshot :636`、`validateSnapshotMemoryObject :749`、`cleanupCowSnapshotObjects :783`、`deactivateCowSnapshotObjects :797`
- **CommitSandbox(三级降级)**:`Cubelet/services/cubebox/template_ops.go:30-230+`(编排、成功后 `setRuntimeSnapshotBindingLabels`、失败 `cleanupArtifacts`→`DeleteObject`);`Cubelet/services/cubebox/snapshot_base_memory.go:19`(哨兵 `ErrNoBaseMemoryForIncremental`)、`:41`(`resolveBaseSnapshotID`)、`:72`(`resolveBaseMemoryObject`)、`:86`(`resolveRestoreBaseSnapshotID`)、`:121`(`resolveMemoryObjectFromSnapshotID`)、`:148-243`(`prepareCommitMemoryArtifact` 三级降级 + base/账本同步约束注释 :211-218);绑定标签:`snapshot_runtime_binding.go:52-91`(restore label stamp、opaque 源置 invalid)
- **Pause**:`Cubelet/services/cubebox/pause_cow.go:73+`(`updateWithPauseCow`:空卷 + full、CommitRootfs 在 PauseToSnapshot 之后、destroy 活沙箱)
- **存储操作**:`Cubelet/storage/cow_store_ops.go`(`GetSandboxRootfs :37`、`CommitRootfs :56`、`CommitRootfsFromBuild :72`、`CreateMemoryVolume :85`、`CommitMemoryFromBase :98`);XFS 格式池 reflink 路径:`Cubelet/storage/shell.go:131`(`cp --reflink=always`)

### cubecow(Rust,XFS reflink 引擎)

- `cubecow/src/lib.rs:29`(ReflinkEngine 导出;`Volume` 含 `export_uuid`/`export_status` 跨节点导出字段;另有 `engine::s3::S3Engine`)
- `cubecow/src/engine/reflink.rs`:`:4-47`(布局与 crash 语义注释:vol-A/snap-1=FICLONE(vol-A)、snap-2=FICLONE(snap-1))、`:87`(`const FICLONE: libc::c_ulong = 0x40049409`)、初始化探测 FICLONE 支持 `:131-160`

### 对照:FC 上游与 AgentEnv fork

- FC File 后端 MAP_PRIVATE restore(前提同构):上游 1.16.1 `src/vmm/src/persist.rs:516-526`、`src/vmm/src/vstate/memory.rs:352-362`
- AgentEnv 方式的三处内脏改动与放大链:`kvcache-ai/firecracker v1.15.1-patch`(本地 `/tmp/fc-fork`)`vstate/vm.rs:329-344`、`devices/virtio/block/virtio/io/async_io.rs:89-118,364-377`、`vstate/memory.rs:773-779,833-880` —— 逐环分析见姊妹篇 [20260824T031853Z](20260824T031853Z-agentenv-dirty-accounting-read-redirty-amplification-chain-source-survey.md);AgentEnv 的 mem 层 restore 同为 MAP_PRIVATE mmap(姊妹篇 §4.2),即 §2.2 所述"AgentEnv 原理上也能走 pagemap 路线"的依据
