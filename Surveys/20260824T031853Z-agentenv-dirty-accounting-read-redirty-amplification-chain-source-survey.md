# AgentEnv 脏账语义深挖:"DAMON 回收 → 重读 → async 标脏 → 整批重物化" 放大链 —— 源码逐环 Survey

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T031853Z`
> 缘起:解释 [023535Z 三线成本 survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §4.3 中的一句话——"脏账语义由 fork 自己的 bitmap 定义,于是有了*页缓存被 DAMON 回收 → md5sum 重读 → async 引擎标脏 → 下次 pause 整批重物化*"。本文把这条因果链的**每一环**拆到源码行级,回答"为什么会这样、谁规定的、正确性依据是什么、哪里可以改"。
> 源码证据:
> - **FC fork**:`kvcache-ai/firecracker` 分支 `v1.15.1-patch`(AgentENV 容器内 `deps/firecracker/1.15.1-patch-v1`),本地 clone `/tmp/fc-fork`;
> - **AgentENV**:BareMetal `/root/AgentEnvWorkSpace/AgentENV`(commit `7f4a9b9`);
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

三个标注点即三个"谁规定的":① FC fork 的 async IO 引擎;② FC fork 的私有 API 的 preserve 语义;③ AgentENV 的物化路径。下面逐环展开。

---

## 2. 环 ①:为什么"读也标脏"——async 引擎的 mark_dirty

### 2.1 源码(FC fork `src/vmm/src/devices/virtio/block/virtio/io/async_io.rs`)

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

而**同步引擎完全没有这个调用**:`sync_io.rs` 全文 grep `mark_dirty` 零命中。也就是说这是 Async 引擎专属行为,而 AgentENV 用户盘固定 `IoEngine::Async`(`AgentENV src/sandbox/firecracker/sandbox.rs:1721-1727`)。

`mark_dirty` 落到 `GuestMemoryMmap`(FC fork `src/vmm/src/vstate/memory.rs:773-779`):

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

### 2.2 为什么 fork 要这么做(设计动机推断)

VMM 把数据**写进** guest 物理内存(`mem.write_slice`)这件事,guest 的 EPT dirty log **未必能记录**:KVM dirty log 通过 EPT 写保护 + PML 记录的是"经过 guest 页表视角的写";异步 io_uring 路径下,VMM 进程在自己的映射上直接写、且写发生在 `KVM_RUN` 之外,是纯宿主侧写。**若不打位,设备 DMA 灌入的数据会在下一次 diff 快照里丢失**——这是正确性必需的。

问题在于实现把它做成了**无差别**:函数名叫 `mark_dirty_mem_and_unwrap`,对"VMM 写入 guest"(读请求的 bounce buffer 拷贝)打位是对的;但对**写请求**也打位(此时 VMM 根本没碰 guest 内存,数据是 guest 自己先写好的,EPT 已经记了),这一半是冗余;更关键的是对**读请求**打位意味着——

> **"这块页缓存重新进入 guest 内存"这一事件,被记账为"这块内存脏了"。**

对从未离开过 guest 内存的页,这个记账是保守但无害的;对**曾经被物化、后来内容从未改变**的页,它就是纯放大器。

---

## 3. 环 ②:preserve 语义——为什么脏位"只增不减"跨窗口累积

### 3.1 源码(FC fork `src/vmm/src/vstate/vm.rs:329-344`)

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

### 3.2 这个语义是特性不是 bug(对"挂起-继续运行"场景)

`preserve` 的注释说明了意图:外部消费者(AgentENV)拿到 ranges 后**可能失败**(process_vm_readv 出错、磁盘满、上层放弃本次 pause)。如果取走即清,失败窗口内的脏页就永久丢了。preserve + 累积保证了"**至少一次**"物化:页一旦被打位,迟早会被某次成功的 pause 物化进 mem 层,物化完成、resume 换新 FC 进程,位图随旧进程消失,账自然重置。

代价:脏账的语义从"自上次成功 checkpoint 以来写过的页"变成了"**自 FC 进程启动以来 VMM 记录过位的所有页**"——两式中环 ① 把"记录过位"扩到了读。

### 3.3 AgentENV 消费侧(物化)

`GET /vm/dirty-memory-ranges` → HVA → `process_vm_readv` 直接读 FC 进程内存 → 512B 扇区 SegmentMapping → overlaybd commit 层(`AgentENV src/sandbox/firecracker/overlaybd_snapshot.rs:717-784`)。环 ①② 攒下的账在这里一次性兑现为 **pause 时间 + 磁盘字节**。

---

## 4. 环 ③:为什么重读一定会发生——DAMON 只回收文件页

### 4.1 源码(AgentENV `src/sandbox/firecracker/config.rs:30-46`)

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

三个要点:

1. **guest 默认开 DAMON reclaim**。这是 AgentENV 有意为之:guest(1–4 GiB 内存)跑 agent 负载,页缓存无界增长会撑爆 mem 层,所以用 DAMON 控制驻留集。参数很保守(冷 60 s 才回收、每秒至多 1 GiB、水位 70‰ 触发),但**一定会发生**——只要沙箱活得够久或内存水位够低。
2. **`skip_anon=Y`:只回收 file-backed 页**。匿名页(/dev/shm、堆)不回收——这保护了匿名内存线(023535Z §4.2 shm 变体 pause#2 平坦的原因),但把全部回收压力集中在**页缓存**上:guest 读过的每一个文件(repo 代码、模型输出、上一次 dd 的大文件)都是候选。
3. 回收一个干净的页缓存页 = 丢弃内容(磁盘上有权威副本)。**下一次任何人读它 = 重新发起块设备读**。guest 内核不知道 host 侧的块设备背后是 reflink/overlaybd,更不知道 host 侧有人会把"读"记账成"脏"。

### 4.2 三环咬合的完整时间线(file-1GiB 因果实验)

`/root/AgentEnvWorkSpace/redirty_test.sh`,单沙箱,`dd 1GiB urandom → /root/f.bin conv=fsync`:

| 步骤 | guest 侧发生 | host 侧账本 | pause 耗时 |
|---|---|---|---:|
| dd 完成 | f.bin 数据在页缓存 + 已写盘 | 写请求完成 → 环① 打位(1GiB) | — |
| **pause#1** | — | 物化 1GiB → mem 层 | **817 ms** |
| resume + 静置 | 无任何 exec | 新 FC 进程,位图从零;页缓存若被 DAMON 回收,静默发生 | — |
| **pause#2**(无 exec) | — | 位图空 → 只物化 ~3 MB | **57 ms** |
| resume + `md5sum /root/f.bin` | 页缓存(可能)已被回收 → 每个块触发读 IO | **读完成 → 环① 再打位 1GiB** | — |
| **pause#3** | — | 再物化 1GiB → 新 mem 层 | **826 ms** |

注意 md5sum 是**纯读、零写入、guest 内存内容零变化**的操作——pause#3 的 826 ms 和 1GiB 磁盘增量全部花在搬运"和 pause#1 时一模一样的字节"上。放大倍数 = 重读率:agent 负载恢复后扫 repo、重读历史输出,放大率轻松到 100%。

### 4.3 什么条件下链路不咬合

| 条件 | 后果 | 证据 |
|---|---|---|
| 用户盘用 Sync 引擎 | pop 路径无 mark_dirty,读不记账 | sync_io.rs 零命中 |
| 不开 DAMON 回收 | 页缓存常驻,重读不触发块 IO | 参数关闭即可(但 mem 层会涨) |
| 重读的页还在页缓存里 | 不发块请求,环①不触发 | pause#2 平坦的原因 |
| 换全新 FC 进程后只写不读 | 只有真实写在记账 | shm 变体 pause#2 52–58 ms |
| gVisor(PR#14228)对照 | 匿名线只按**写**序列化,读不计脏 | 023535Z §5.2 |

**三环缺一不可**——这是它能同时通过 133500Z(shm-512M 仅首轮慢)与 041720Z(file-1G 三轮全慢)两批数据的检验、又被 redirty_test.sh 单变量因果实验钉死的原因。

---

## 5. 正确性与修复空间分析(如果 AgentEnv 要改)

这个放大链**不产生错误结果**——物化的页内容正确,只是浪费。修复按侵入度排序:

1. **读路径按需打位(治本,改 FC fork 一处)**:`mark_dirty_mem_and_unwrap` 只在 `bounce_buf.is_some()`(VMM 实际写入了 guest 内存)时 `mark_dirty`。写请求分支删掉是安全的吗?——guest 写盘前数据已在 guest 内存,EPT 已记脏;VMM 侧不写 guest。唯一要审计的是 vMMIO/间接描述符路径是否依赖此处兜底。预期效果:读重放完全不再入账,pause 回到 O(真实写集)。
2. **preserve 换成 ack 式清除(治本,协议改动)**:`dirty-memory-ranges` 返回后,由 AgentENV 在物化成功后显式调一个 `clear-dirty-ranges` 确认。保留"消费者失败不丢账"的意图,消掉单调累积。改动跨 FC fork + AgentENV 两处。
3. **DAMON 参数收紧(缓解,零代码)**:调高 `wmarks` 或加大 `min_age`,减少回收→重读频率。代价是 guest 驻留集变大、mem 层首轮更大——只是把成本从"重复物化"挪回"首次物化"。
4. **mem 层内容寻址去重(缓解,AgentENV 侧)**:物化时按页/段哈希,命中已有 managed-layer 则引用而非重写。读重放的页与上一轮内容相同,大概率全命中——把 826 ms 降为哈希计算时间(但 O(读集) 的扫描仍在)。

对读密集 agent 负载,1+2 组合后 AgentEnv 的 pause 语义才与 CubeSandbox soft-dirty(只记真实写,见姊妹篇)和 gVisor(只序列化写过的页)对齐。

---

## 6. 证据索引

### FC fork `v1.15.1-patch`(本地 `/tmp/fc-fork`)

- 读也标脏:`src/vmm/src/devices/virtio/block/virtio/io/async_io.rs:89-118`(esp. 115 `mem.mark_dirty(addr, count as usize)`),唯一调用点 `:374`(pop 路径);`sync_io.rs` 无 `mark_dirty`(grep 零命中)
- preserve 语义:`src/vmm/src/vstate/vm.rs:329-344`(`get_dirty_memory_ranges_preserve`,含"so a later snapshot still sees the pages if the external consumer fails"原注释)
- 内部位图实现:`src/vmm/src/vstate/memory.rs:773-779`(mark_dirty)、`:833-858`(dirty_memory_ranges = KVM ∪ 内部)、`:860-880`(store_dirty_bitmap,OR 回);清零点仅 `dump_dirty` 成功后(上游语义保留)
- state-only 快照不触发清零:`src/vmm/src/persist.rs:178`(`if let Some(mem_file_path)`)
- 上游对照(内部位图本源只记 vMMIO):上游 1.16.1 `memory.rs:460-478`,见 [023535Z survey](20260824T023535Z-cr-vs-guest-memory-fc-agentenv-gvisor-cubesandbox-anon-rwfs-line-survey.md) §3.2

### AgentENV(BareMetal `/root/AgentEnvWorkSpace/AgentENV`,commit `7f4a9b9`)

- DAMON boot args(含 `skip_anon=Y`):`src/sandbox/firecracker/config.rs:30-46`(本文 §4.1 全文引)
- 用户盘 Async 引擎:`src/sandbox/firecracker/sandbox.rs:1721-1727`
- 物化路径:`src/sandbox/firecracker/overlaybd_snapshot.rs:717-784`(process_vm_readv → 512B segment → overlaybd commit)
- pause 链与 state-only Diff:`src/sandbox/firecracker/sandbox.rs:660-679,831-837`、`instance.rs:457-464`

### 实验(BareMetal,2026-08-24)

- 因果实验:`/root/AgentEnvWorkSpace/redirty_test.sh`(817/57/826 ms,file-1GiB)
- scaling 与假阴性澄清:`aenv_anon_scale.sh`、`shm512_verify.sh`(023535Z §4.2 表)
