# CubeSandbox 宿主 pagemap / soft-dirty 脏页采集机制源码 Survey —— 兼"自维护 FC fork 该走哪条路"的推荐

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T033000Z`
> 缘起:回答两个问题——(1) CubeSandbox 的宿主 `/proc/<pid>/pagemap` 和 soft-dirty 位到底是怎么拿到的;(2) 如果我们自己维护一个 Firecracker fork,增量内存快照的记账应该学 AgentEnv(KVM dirty log + VMM 内部位图 + 私有 API 外部采集)还是学 CubeSandbox(纯 `/proc/self` 页表自省)?
> 源码证据:BareMetal `/root/CubeSandboxWorkSpace/CubeSandbox`(Tencent 开源,Cloud Hypervisor fork),关键文件已取回本地 `/tmp/cube-src/`:
> - `hypervisor/vmm/src/soft_dirty.rs`(846 行,含单测)
> - `hypervisor/vmm/src/pagemap_anon.rs`(467 行,含单测)
> - `hypervisor/vmm/src/memory_manager.rs:2360-2460`(`send_pagemap_anon_memory`)、`:2458-2650`(`send_soft_dirty_memory`)
> 姊妹篇:[AgentEnv 读重脏放大链 survey](20260824T031853Z-agentenv-dirty-accounting-read-redirty-amplification-chain-source-survey.md)(环①②③ 的对位分析)、[023535Z 三线成本 survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §6

---

## 1. 结论摘要

1. **CubeSandbox 没有碰任何 KVM 接口,也不需要跨进程读别人**——它的 VMM 进程把 guest RAM 以 `MAP_PRIVATE` mmap 形式放在**自己的地址空间**里(restore 即 mmap 快照文件),所以"guest 写了哪些页"这个问题等价于"**我自己的哪些页被 CoW 成匿名页/带上了 soft-dirty 位**",答案就躺在 `/proc/self/pagemap` 里,一次 `seek + read_exact` 批量拿走。
2. 两条互补的判定线:**pagemap-anon**(bit63 present + PFN → `/proc/kpageflags` 查 `KPF_ANON`,累计式,"自 restore 以来 CoW 过的所有页")与 **soft-dirty**(`clear_refs(4)` 布防 + pagemap bit55,真窗口 delta)。CubeSandbox 稳态用**二者交集**:只有"确实是 CoW 匿名页(内容才可能偏离 base 文件)且本窗口写过(才是本次必须记的 delta)"的页才写入快照。
3. 写入侧不产 raw delta:目标文件是**全尺寸 base 镜像的原地覆写**(base 由 CubeCoW/XFS FICLONE 预先 reflink-clone,覆写只破坏自己的 extent),消费方(restore)拿到的永远是一份自洽的 full 镜像——**记账是增量的,格式是全量的**。
4. **推荐**:自维护 FC fork 走 **CubeSandbox 路线为骨架**(零 KVM API 改动、零私有 dirty API、FC 的 File 后端本来就是 `MAP_PRIVATE` mmap、语义上读中性、天然衔接 reflink 代际),**吸收 AgentEnv 的两点**:state-only 快照分解(vm_state 与内存分离)与 KVM dirty log 作为大内存 VM 的可选加速/校验通道。理由与边界见 §6。

---

## 2. 前置:为什么"读自己的 /proc/self/pagemap"就能给 guest 内存记账

Cloud Hypervisor(以及 Firecracker)的 guest 内存模型:VMM 在自己进程里为每个 memslot 建一段映射,再把该映射的 host 物理页经 `KVM_SET_USER_MEMORY_REGION` 塞给 KVM 当 guest RAM。**guest 的每一次内存写,物理上就是 CPU 对这段 VMM 进程映射的写**(经由 EPT 直写,不经 VMM 代码)。

CubeSandbox 的 restore 是 fast-restore 路径:把快照文件 fd 以 `MAP_PRIVATE` mmap 为 guest RAM(`memory_manager.rs:1326-1376,1509-1516`,与 FC 上游 File 后端 `persist.rs:516-526` → `memory.rs:352-362` 同构)。于是内核页表视角出现一个干净的二分:

| guest 对某页做过什么 | 内核页表状态 | pagemap 表现 |
|---|---|---|
| 从未触碰 | 无 PTE | present=0 |
| 只读(页错误读入,未写) | 映射到 base 文件的 page cache | present=1,KPF_ANON=0 |
| **写过** | **CoW 出私有匿名页** | present=1,**KPF_ANON=1**(且写后 bit55=1) |

**"guest 写集合"被内核免费记在了页表里**——这就是全部魔法的来源。VMM 不需要 ptrace 别人、不需要 KVM dirty log、不需要 fork 内核;它只是**自己 mmap 了一块大内存,然后问内核"我这块内存现在长什么样"**。`/proc/self/pagemap` 按 *8 字节/页* 索引(host 虚拟页号),`File::open` + `seek` + `read_exact` 一次批量读完整个区域(`pagemap_anon.rs:198-226`、`soft_dirty.rs:276-297`)。

一个必要的权限细节:pagemap 的 bit63(present)/bit62(swap)/bit55(soft-dirty)对进程自身**始终可读**;但 **PFN(bits 0–54)与 `/proc/kpageflags` 只有 `CAP_SYS_ADMIN` 可读**(无权时 PFN=0)。所以 soft-dirty 线完全不需要特权,而 **pagemap-anon 线(要拿 PFN 查 kpageflags)需要 VMM 以 root/保留该 capability 运行**——代码里显式处理了无权情形(`pagemap_anon.rs:253-258` 返回 `NoCapSysAdmin` 错误)。

---

## 3. 线一:pagemap-anon(累计式"自 restore 以来写过的匿名页")

### 3.1 判定逻辑(`pagemap_anon.rs:188-286`)

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

粒度细节:全部下标/偏移用 `host_page_size()`(`sysconf(_SC_PAGESIZE)`,probe 一次缓存,`pagemap_anon.rs:36-47`)——注释明确是为了 **ARM64 64KiB 页**内核不至错位索引,这是一个很容易被忽略的坑。

### 3.2 累计性(为什么需要第二条线)

anon 是"**自 restore 以来的并集**":页一旦 CoW,永远是匿名页,哪怕此后再没改过。deep-dive 实测(200 MiB 沙箱增量逐次 200 MiB→…→15 GiB)就是这条单调曲线。它正确(内容确实可能偏离 base)、但不是 delta——长寿命沙箱每个快照都要重写整个累计集。

## 4. 线二:soft-dirty(真窗口 delta)

### 4.1 布防与采集(`soft_dirty.rs:224-326`)

```rust
const CLEAR_REFS_SOFT_DIRTY: &[u8] = b"4\n";
const PAGEMAP_SOFT_DIRTY_BIT: u64 = 1 << 55;

pub fn clear_soft_dirty() -> Result<()> {
    // 写 "4" 到 /proc/self/clear_refs:
    // 清掉本进程每个 PTE 的 soft-dirty 位,并给每个 VMA 重新挂上 VM_SOFTDIRTY
    // (之后任何写都会以 fault 方式重新装 PTE 并置位 bit55)
}

pub fn get_soft_dirty_pages(host_addr: u64, length: u64) -> Result<Vec<bool>> {
    // 同样 seek + read_exact 批量读 pagemap:
    //   swapped → true(保守)
    //   present && bit55 → true(本窗口写过)
}
```

窗口协议:**首次快照写 full/anon 集 → `clear_refs(4)` 布防 → 此后每个写页 fault 时置 bit55 → 下次快照只收 bit55=1 的页 → 再布防**。配套单测(`soft_dirty.rs:616-750`)明确断言了 delta 语义:round2 写页 2/3/4 → 只回这三页;round3 只写页 7 → **只回页 7,不回 2/3/4**。

### 4.2 能力探测的 round-trip 技巧(`soft_dirty.rs:121-215`)

写 "4" 到 clear_refs 在 `CONFIG_MEM_SOFT_DIRTY=n` 的内核上**也会成功(静默 no-op)**,返回值不可信。探测改用真实往返:mmap 一页匿名页 → 写一字节(首写装 PTE,若内核支持则 VMA 的 VM_SOFTDIRTY 传播到 PTE bit55)→ 读 pagemap 看 bit55 是否为 1。**只有确认支持才做那次昂贵的全局布防**,不支持时连 mmap_lock 写锁的 PTE 全量遍历都省掉,调用方静默降级。

### 4.3 为什么要交集而不是单用 soft-dirty(`soft_dirty.rs:399-490`)

源码注释把三种近似的偏差说得很清楚:

- **anon-only**:正确但累计(§3.2);
- **soft-dirty-only**:两个污染源——(a) host 侧落在 file-backed page cache 上的写(内容与 base 磁盘副本相同,重存纯浪费);(b) 刚布防完的第一个窗口里,任何装 PTE 的写(包括 fault 进非匿名页的)都会置位;
- **anon ∩ soft-dirty**:恰好 = "内容可能偏离 base ∧ 本窗口真的改过",既最小又安全。

```rust
let must_save: Vec<bool> = anon_pages.iter().zip(dirty_pages.iter())
    .map(|(&is_anon, &is_dirty)| is_anon && is_dirty).collect();
```

## 5. 编排与写入侧(`memory_manager.rs:2360-2650`)

### 5.1 首快照的惰性布防(`send_soft_dirty_memory:52-79`)

不在 boot/restore 时布防(`clear_refs` 是全 PTE 写锁遍历,multi-GiB guest 上数百 ms,是 pause 尖峰的主导项),而是**第一次快照时**:写一份 anon 全集 → `probe_soft_dirty_support()`(成功即含布防)→ 置 `soft_dirty_armed`。稳态路径(`:85-156`):交集过滤 → 原地覆写 → **每轮末尾再 `clear_soft_dirty()` 重布防**(注意:这次布防的 PTE 遍历是每窗口重复发生的,不是一次性成本)。任何失败(内核不支持、clear 失败)→ 降级 anon 路径,`armed=false`,下轮重试。

### 5.2 原地覆写:增量记账、全量格式

两条增量线都要求目标 base 文件存在(不存在则显式报错让调用方先 Full),然后:

```
gpa_to_file_offset 表(线性布局)→ 对每个 filtered range:
    save_range_to_file(memory_file, range, file_off)   // pread guest 内存 → pwrite 到偏移
```

产物是**全尺寸、自洽的 full 镜像**——VMM 其余部分(restore、迁移接收端)完全无感知,不需要"base+delta 链"的恢复协议。覆写之所以便宜且不破坏上一代,是因为 Cubelet 在快照前已用 XFS FICLONE 把旧 base reflink-clone 过(`Cubelet/services/cubebox/appsnapshot.go:570-583`),覆写只 CoW 掉自己的 extent——**"增量写放大"被 reflink 吸收,与 PR#14228 clone-on-adopt 同一模式**。restore 侧 `MAP_PRIVATE` mmap 近 O(1),写时内核 CoW,回到 §2 的前提,闭环。

### 5.3 每次快照的成本清单

| 项 | 复杂度 | 备注 |
|---|---|---|
| pagemap 批量读 | O(区域页数) 顺序读(8B/页,≈2 ms/GiB) | 每轮 ×2(anon + softdirty 各一次读) |
| kpageflags 逐 PFN 查 | O(present 页数) **随机 seek** | anon 线独有;最值得优化的点(可按 PFN 升序排序化随机为顺序) |
| `clear_refs(4)` 布防 | O(全部 PTE),mmap_lock 写锁 | **每窗口重复**;multi-GiB 上数百 ms,是 pause 的主导项(源码注释自认) |
| 布防后首个写页 | 每页一次 fault | 与 KVM dirty log 的 EPT 写保护同型,摊在运行期 |
| 覆写 | O(交集脏页) 顺序写 | reflink 保护上一代 |

---

## 6. 推荐:自维护 FC fork 学哪条路

### 6.1 两条路线的本质差异

| | **AgentEnv 方式**(KVM dirty log + VMM 内部位图 + 私有 API) | **CubeSandbox 方式**(pagemap/soft-dirty 自省) |
|---|---|---|
| 脏信息来源 | KVM `GET_DIRTY_LOG`(EPT/PML 硬件记账)+ FC 内部 `AtomicBitmap`(VMM 侧事件,**读完成也打位**) | 内核页表本身(CoW→KPF_ANON;bit55 soft-dirty),**读中性** |
| 对 FC/KVM 的侵入 | 必须 fork FC:私有 `GET /vm/dirty-memory-ranges`、state-only 快照、bitmap 语义改造;且 async/sync 引擎行为不一致(姊妹篇环①) | **零 KVM API 改动、零新增私有 dirty API**;只要有一个"按 range pwrite"的导出口 |
| 前提条件 | dirty tracking 需在 guest 内存创建时开启(EPT 写保护常驻) | restore 必须是 `MAP_PRIVATE` mmap(**FC File 后端本来就是**,零改造);anon 线要 `CAP_SYS_ADMIN` |
| 增量语义 | 内部位图 preserve+累积(进程生命周期单调),"至少一次"物化 | anon 累计 ∩ soft-dirty 真 delta,窗口干净 |
| 每轮固有开销 | PML 硬件记账(近零)+ O(dirty) 收割 | `clear_refs` 全 PTE 遍历(每轮,数百 ms@multi-GiB)+ kpageflags 随机读 |
| 外部采集 | `process_vm_readv` 跨进程读 FC 内存(AgentENV 是父进程所以有权限) | 无(VMM 自己 `pread` 自己的映射) |
| 产物形态 | overlaybd mem 层链(需 compact/继承协议) | 全尺寸自洽镜像 + reflink 基座(消费方零感知) |
| 降级路径 | 无(私有协议,一坏全坏) | soft-dirty→anon→Full 三级,探测自动降级 |
| 跨节点 | 层链整体搬迁 | base(可能已 FICLONE 共享)+ delta 覆写,与 reflink 存储天然组合 |

### 6.2 结论:CubeSandbox 为骨架,AgentEnv 为补丁

**推荐以 CubeSandbox 方式为主**,四个理由:

1. **前提免费**:FC 的 File 后端 restore 就是 `MAP_PRIVATE` mmap(`persist.rs:516-526` → `memory.rs:352-362`),CubeSandbox 的整个模型(写=CoW=anon=bit55)在 FC 上原样成立,一行内存管线都不用改。而 AgentEnv 方式要动 KVM 交互、bitmap 语义、IO 引擎三处 FC 内脏,每处都是长期维护负担(async/sync 不一致就是前车之鉴)。
2. **语义正确**:读中性 + 真 delta + 窗口重置,直接避免姊妹篇整条"DAMON 回收→重读→标脏→重物化"放大链的账目污染(guest 内核把换出的页缓存重读进 guest RAM 的那次写当然仍会记——但那是真实写,任何系统都躲不掉;被消灭的是"重放读的 DMA 目标页被无差别入账"这类虚增)。
3. **生态咬合**:原地覆写 + FICLONE 基座 = 与我们 PR#14228 的"生命周期绑定、内容不进 image"哲学同构;跨节点搬 base+delta、代际共享、clone(n) 全部顺下来,不需要 OverlayBD 层链那套 compact/32 层上限协议。
4. **可降级、可测试**:soft-dirty→anon→Full 探测降级 + 全套页级单测(round-trip delta 断言)是可以带着走的工程资产。

**同时吸收 AgentEnv 的三点**(都是低成本高收益):

- **state-only 快照分解**(`vm_state.bin` 与内存 artifact 分离,~20KB 控制态):这个 fork 改动极小且与任何内存方案正交;
- **KVM dirty log 作为大内存可选通道**:multi-GiB guest 上 `clear_refs` 每轮数百 ms 的 PTE 遍历会重新变成 pause 主导项,而 PML 硬件记账近零——可以做成"小内存 pagemap / 大内存 dirty log"的双模,或用 KVM log 做交集校验(两个独立来源的交集是极强的正确性防线);
- **preserve/ack 的教训**(而非其实现):外部消费者可能失败,账目清零必须由消费成功的 ack 驱动,而不是"取走即清"(丢账)或"永不清"(单调累积)——CubeSandbox 的"布防在写盘成功之后"(`memory_manager.rs:146-154`,覆写失败则 disarm 并降级)已经是这个语义,照抄即可。

**两个必须提前处理的坑**:

1. **`CAP_SYS_ADMIN`**:FC production 用 jailer 降权运行,pagemap PFN/kpageflags 会变 0/不可读 → anon 线失效。要么保留该 capability,要么实现纯 soft-dirty 模式(bit55/present 自身可读,无需特权;代价是失去 anon 交集,需按 CubeSandbox 注释处理 file-backed 写污染),要么 fallback 到 Full/`clear_refs`+mincore。
2. **布防时机**:抄"首次快照惰性布防 + 每轮末尾重布防",并把 `clear_refs` 耗时打进 pause 分相日志(CubeSandbox 特意在 info 级打了这条日志);若 pause 尖峰不可接受,退化为 anon 累计 + 周期性 Full 重置(CubeSandbox Incremental 模式就是这个姿态)。

---

## 7. 证据索引

### CubeSandbox(BareMetal `/root/CubeSandboxWorkSpace/CubeSandbox`,本地副本 `/tmp/cube-src/`)

- pagemap-anon 判定:`hypervisor/vmm/src/pagemap_anon.rs:188-286`(bit63/62、PFN mask `(1<<55)-1`、KPF_ANON=bit12、swapped→true、NoCapSysAdmin);host page size:`:36-47`(ARM64 64KiB 防坑);合并:`coalesce_pages_to_ranges :58-96`
- soft-dirty:`hypervisor/vmm/src/soft_dirty.rs:224-249`(`clear_refs "4"` + 耗时 info 日志 + "dominant snapshot-time stall" 自认)、`:267-326`(bit55 采集)、`:121-215`(probe round-trip)、`:399-490`(anon∩soft-dirty 交集 rationale 与实现);delta 语义单测:`:616-750`;交集单测(含 CAP_SYS_ADMIN skip):`:765-845`
- 编排:`hypervisor/vmm/src/memory_manager.rs:2360-2460`(`send_pagemap_anon_memory`:base 必须存在 + 原地覆写 + gpa→file offset)、`:2458-2650`(`send_soft_dirty_memory`:惰性布防、交集、覆写后重布防、失败 disarm 降级)、`:2475-2479`(布防成本注释)
- fast restore mmap / CubeCoW 预 clone / 三模式分派:见 [023535Z survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §6 与其证据索引(`memory_manager.rs:1326-1376,1509-1516`、`Cubelet/services/cubebox/appsnapshot.go:570-583`、`vm-migration/src/lib.rs:110-123`)

### 对照:FC 上游与 AgentEnv fork

- FC File 后端 MAP_PRIVATE restore(前提同构):上游 1.16.1 `src/vmm/src/persist.rs:516-526`、`src/vmm/src/vstate/memory.rs:352-362`
- AgentEnv 方式的三处内脏改动与放大链:`kvcache-ai/firecracker v1.15.1-patch`(本地 `/tmp/fc-fork`)`vstate/vm.rs:329-344`、`devices/virtio/block/virtio/io/async_io.rs:89-118,364-377`、`vstate/memory.rs:773-779,833-880` —— 逐环分析见姊妹篇 [20260824T031853Z](20260824T031853Z-agentenv-dirty-accounting-read-redirty-amplification-chain-source-survey.md)
