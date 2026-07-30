# AgentENV 缺少 vCPU 弹性的影响与 Firecracker 运行态内存部分回收机制分析

> 调研时间：2026-07-28 17:01:35（UTC+8）  
> AgentENV 源码版本：`6296bc4be7ad79eb3a278eb5264ef011c341adf5`  
> AgentENV bundled Firecracker：`1.15.1-patch-v1`  
> 源码目录：`/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV`  
> 关联文档：[AgentENV Firecracker Idle CPU/内存释放与恢复机制源码分析](/mnt/u/hukeyang/AKernel/AKernel_Docs/Surveys/20260728T073510Z-agentenv-firecracker-idle-cpu-memory-survey.md)

## 1. 直接结论

### 1.1 vCPU

当前 AgentENV 不支持 vCPU hotplug，也不支持把一个正在运行或已暂停的 1-vCPU microVM 原地扩为 4 vCPU：

- `cpuCount` 只在 cold create 时进入 `FirecrackerSandboxConfig`。
- Firecracker `/machine-config` 的 `vcpu_count` 只能在 pre-boot 阶段设置。
- snapshot 中已经序列化了 vCPU topology、寄存器和 CPU feature state，resume 不会重新配置 CPU 数。
- snapshot-based template 明确拒绝修改 CPU/内存规格。
- bundled Firecracker API 有 `/hotplug/memory`，没有对应的 CPU hotplug endpoint。

缺少这项能力的主要问题不是“4 个空闲 vCPU 会一直烧掉 4 个物理核”——没有 runnable guest work 时 vCPU 线程几乎不消耗 CPU 时间——而是以下两难：

```text
按常态配 1 vCPU：burst 阶段没有 guest 内并行度，任务和尾延迟变长
按峰值配 4 vCPU：AgentENV 始终按 4 CPU 计逻辑 reservation，调度密度降低
放宽 overcommit：提高平均密度，但同步 burst 时产生争用和 noisy neighbor
```

### 1.2 运行态内存部分释放

Firecracker 支持在 VM 不 pause、不 snapshot、不终止进程的情况下部分释放 host 内存。需要区分三种机制：

| 机制 | VM 是否继续运行 | guest 可见容量是否变化 | 是否能降低 host RSS | AgentENV 当前状态 |
|---|---|---|---|---|
| free-page reporting | 是 | 否 | 是，只释放 guest 已空闲页 | 已启用 |
| traditional balloon target | 是 | topology 不变，但有效可用量下降 | 是，可主动要求回收部分页 | Firecracker 支持，AgentENV 未控制 |
| virtio-mem hotplug | 是 | 是，动态 plug/unplug hotplug pool | 是，unplug 后按 block 释放 backing | Firecracker 支持，AgentENV 未接入 |
| pause + snapshot + stop | 否，期间中断 | VM runtime 整体不存在 | 最彻底，释放整个 VMM 私有 RSS | AgentENV 已实现 |

因此，对用户问题最精确的回答是：

> AgentENV 当前确实支持运行态“部分释放内存”，实现是 `virtio-balloon free_page_reporting`。它把 guest 已经空闲的页对应的 host backing 丢弃，从而降低 Firecracker RSS，但不会缩小 guest `MemTotal`，也不会改变 AgentENV 的 `memory_mib` reservation。Firecracker 还支持更主动的 balloon inflation 和真正改变 guest online memory 的 virtio-mem，但 AgentENV 目前没有接入这两个控制面。

## 2. 当前 vCPU 为什么是固定的

### 2.1 创建阶段一次性确定

cold sandbox API 读取 `cpuCount`，没有提供运行时 resource patch；请求值被保存为 `SandboxResources.cpu_count`（[sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/api/impls/sandbox.rs:252)）。factory 随后将其写入 fresh Firecracker config（[factory.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/factory.rs:100)）。

VM 启动前调用：

```rust
set_machine_config(mem_size_mib, vcpu_count, false, false)
```

调用点见 [sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:1543)，底层向 `/machine-config` 发送 PUT（[instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:275)）。

bundled Firecracker schema 明确将 `/machine-config` 标为 `Pre-boot only`，vCPU 范围为 1～32；SMT 开启时还要求 1 或偶数（[firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:397)）。AgentENV API 自身目前只检查 `cpuCount > 0`，因此超过 Firecracker 上限的请求会在更下游失败，而不是 API 层立即拒绝。

### 2.2 Resume 不能作为离线 resize

AgentENV resume 在一个新的 Firecracker 进程中直接加载 `vm_state.bin` 和 memory backend，随后恢复 VM；它不会先加载 snapshot 再调用 `/machine-config`。Firecracker snapshot load 本身也要求在 fresh process、尚未配置其他 VM resource 时执行。

源码还明确说明：CPU config 只应在第一次启动前应用；恢复时完整 CPU state 已在 `vm_state.bin`，重新应用会被 Firecracker 拒绝（[config.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/config.rs:484)）。template builder 同样明确拒绝 snapshot-based build 改变 CPU 或 memory（[builder.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/template/builder.rs:234)）。

所以以下路径不可行：

```text
1-vCPU Running
  -> Pause/Snapshot
  -> 修改 metadata.cpu_count=4
  -> Resume 成 4-vCPU
```

metadata 数字即使被修改，也不会改变 snapshot 中的 vCPU topology，反而会导致控制面计量与真实 VM 不一致。

## 3. 没有 vCPU 弹性会带来什么问题

### 3.1 Agent 工作负载天然相位化

典型 Agent sandbox 会在多种阶段之间快速切换：

- 等待 LLM、网络 API、用户输入时接近 CPU idle。
- 解析代码、建立索引、运行语言服务时短时需要多核。
- 编译 Rust/C++、运行测试、压缩镜像层时可能持续吃满多个核。
- 多 tool 或多 shell command 并发时需要 guest scheduler 同时运行多个进程。
- 浏览器、数据库、编译器与 agent runtime 并存时，交互线程容易被后台任务阻塞。

固定 1 vCPU 会把所有 runnable task 放进一个 guest run queue。即使宿主还有大量空闲物理核，这个 VM 也无法利用，因为 guest 只看到一个 CPU。

### 3.2 低配会放大执行时间和尾延迟

若创建时只配置 1 vCPU：

- 多线程编译和测试退化为近似串行。
- 一个 CPU-bound build 会阻塞 agent 控制进程、envd 和交互命令。
- run queue 增长，p95/p99 比平均延迟恶化更明显。
- 更容易触发 tool timeout、sandbox TTL 或上层请求 timeout。
- 单 VM 无法通过“节点还有空闲 CPU”自动加速。

理想情况下，一个完全可并行的 4-core burst 在 1 vCPU 上的墙钟时间可接近 4-vCPU 的四倍；真实程序受串行段、锁和 I/O 限制，实际加速低于 4 倍，但固定 1 vCPU 仍构成不可突破的并行宽度上限。

### 3.3 峰值预配不会持续烧核，但会浪费逻辑容量

另一种做法是所有 VM 一开始就配置 4 vCPU。空闲 VM 的四个 vCPU thread 没有 runnable work 时不会持续占用四个物理核；因此不能把它理解为始终浪费 300% CPU cycles。

真正的问题是 AgentENV 资源模型始终把这个 VM 记作 4 个 allocated CPU。除 `Paused` 外，所有 active/transition state 都按 `resources.cpu_count` 计量；Paused 则把原规格记入独立 reservation（[metrics.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/orchestrator/metrics.rs:56)）。

例如一个 64-core node：

| 策略 | 每 VM 声明 vCPU | 严格 reservation 下 VM 数 | 单 VM burst 并行度 |
|---|---:|---:|---:|
| 常态配置 | 1 | 64 | 1 |
| 峰值配置 | 4 | 16 | 4 |
| 4 倍 overcommit | 4 | 64 | 4，但同时 burst 时 256 vCPU 争 64 core |

这个例子忽略 server/daemon 开销，只用于说明 reservation 与 burst capacity 的矛盾。

再考虑一个示例负载：90% 时间需要 1 CPU，10% 时间需要 4 CPU，则平均 demand 是 `0.9×1 + 0.1×4 = 1.3 CPU`。若固定预配 4 vCPU，按 reservation 计算的平均利用率只有 `1.3/4 = 32.5%`，reservation 是平均 demand 的约 3.08 倍。这同样只是模型示例，不是本项目 benchmark 数据。

### 3.4 Overcommit 会把浪费变成争用风险

控制平面配置明确允许 CPU allocated percent 超过 100%，因为 reservation 支持 overcommit（[config.go](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/services/shared/config/config.go:35)）。overcommit 可以提高平均密度，但无法创造物理核：

- 多个 agent 同时进入 build/test burst 时，宿主 runnable threads 激增。
- context switch、cache thrashing 和 steal/排队时间增加。
- 一个 noisy VM 可影响同节点其他 VM 的 p99。
- 调度器如果只看历史/当前 host CPU usage，可能在同步 burst 前继续放置 VM。
- 如果严格看 `allocated_cpu`，峰值预配又会让空闲节点过早被过滤。

当前 scheduler 同时支持 actual CPU percent 和 allocated CPU percent 阈值（[filter.go](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/services/scheduler/internal/filter.go:28)），但它没有改变 per-VM CPU topology，也没有为单个 Firecracker 进程建立动态 CPU entitlement。

### 3.5 Stateful vertical scaling 无法用横向扩容完全替代

没有 vCPU resize 时只能新建一个 4-vCPU sandbox，然后在应用层迁移状态。对无状态任务可以接受，但对以下状态代价很高：

- 正在运行的进程、线程栈和内存对象。
- 已建立的语言服务索引、编译 cache、JIT 状态。
- 本地临时文件、锁、Unix socket 和进程间通信。
- 正在进行的多步 agent session。

snapshot/fork 可以保留这些状态，但 fork 出来的 VM 继承原来的 CPU/内存 topology，不能把 1-vCPU snapshot 变成 4-vCPU snapshot。为了提供不同 SKU，可能不得不维护 1/2/4-vCPU 多套模板，增加构建、存储、兼容验证和 cache 碎片。

### 3.6 只有全 Pause，没有运行态 CPU 缩容

AgentENV 目前释放一个 running VM 的 vCPU 资源只能走完整 Pause：持久化 snapshot 后 stop Firecracker 进程（[service.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/orchestrator/service.rs:1162)）。这适合真正 idle 的环境，但不能表达：

```text
VM 保持运行、进程和连接不动
guest 仍看到足够并行度
宿主暂时只给它约 0.5～1 core entitlement
burst 到来时放宽到 4 core
```

这也是为什么“动态 CPU quota”可能比真正 CPU hotplug 更适合作为 AgentENV 的第一步 CPU 弹性。

## 4. 更现实的 CPU 弹性方案

### 4.1 峰值 vCPU topology + 动态 cgroup quota

短期可让 guest 从启动时就看到峰值 4 vCPU，再把 Firecracker PID 放入 per-sandbox cgroup：

```text
guest topology: 固定 4 vCPU
平时 cpu.max: 约 1 core
burst 时 cpu.max: 放宽至 4 core
压力结束: 再收回至 1 core
```

优点：

- 不改变 guest CPU topology。
- 不破坏 snapshot 格式和 vCPU state。
- 调整可在运行态快速完成。
- guest 在 burst 时已经具备 4-way parallelism。
- 可同时使用 `cpu.weight` 做相对公平和 `cpu.max` 做硬 entitlement。

限制：

- guest 始终看到 4 CPU，可能据此创建 4 个 worker，即使当前 quota 只有 1 core。
- quota 放宽前，多 worker 会互相争同一个 entitlement。
- 需要真实的 per-sandbox usage、headroom 和并发 resize admission。
- 当前 AgentENV 没有为每个 Firecracker runtime 写 `cpu.max`/cpuset 的实现；现有 cgroup 代码主要用于读取 AgentENV 进程所在 cgroup 的 host capacity。

### 4.2 真正 vCPU hotplug

真正从 guest topology 的 1 CPU 变成 4 CPU，需要 Firecracker 上游支持 vCPU device/model hotplug、APIC/GIC/中断状态、guest CPU online、snapshot schema 和恢复兼容。当前 bundled API 没有该 endpoint，因此这不是只给 AgentENV 增加一条 HTTP route 就能完成的功能。

即使未来 Firecracker 支持，控制面仍需解决：

- resize 前原子预留 node headroom。
- 多 VM 同时扩容时的仲裁。
- guest online/offline 是否成功的可观测性。
- 部分成功和 timeout 回滚。
- snapshot 对不同 CPU topology 的版本兼容。
- 缩容时目标 vCPU 是否仍运行不可迁移任务。

## 5. Firecracker 的四层内存回收语义

理解“部分释放”之前，需要把以下四个量分开：

1. **Configured boot memory**：`machine-config.mem_size_mib`，guest 的基础内存 topology。
2. **Guest effective available memory**：扣除 balloon、内核和 workload 后还能分配多少。
3. **Guest online hotplug memory**：virtio-mem pool 中当前 plugged/online 的容量。
4. **Host resident memory/RSS**：Firecracker 地址空间当前真正有物理 backing 的页，加上 VMM 开销。

这四者不会自动保持相等。

### 5.1 Free-page reporting：不改变容量，只丢弃空闲页 backing

free-page reporting 是 virtio-balloon 的一个 feature。guest balloon driver 持续从 buddy allocator 找到未使用范围并报告给 VMM；Firecracker 对这些范围执行 `MADV_DONTNEED`，使相应 host physical backing 可被回收，Firecracker RSS 下降。

Firecracker v1.15.1 官方文档明确说明：

- reporting 只能 pre-boot 启用。
- 启用后持续运行，不能在运行中停止。
- guest 需要 `CONFIG_PAGE_REPORTING`。
- Linux driver 通常在约 2 秒延迟后开始上报。
- 回收粒度和收益取决于 `page_reporting_order` 及 workload。

参见 [Firecracker ballooning：free-page reporting](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md#L307-L347)。

它的关键语义是：

```text
guest MemTotal 不变
guest 没有失去使用这些地址的权利
当前 free 页的 host backing 被丢弃
guest 以后重新分配/写入时，host backing 再按需 fault 回来
```

因此它是最透明的运行态部分回收，但只能回收 guest 已经 free 的页；仍被进程占用、被 pin 或尚未回收的匿名 heap 不能凭空释放。

### 5.2 Traditional balloon inflation：主动降低 guest 有效可用量

传统 balloon 的 target 由 `amount_mib` 指定。target 增大时，guest balloon driver 在 guest 内分配页并将 PFN 告诉 Firecracker，Firecracker 丢弃对应 host backing；target 减小时 driver deflate，把页还给 guest。Firecracker 官方说明 target 可以在运行中改变（[Firecracker ballooning 基本机制](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md#L3-L18)）。

它与 free-page reporting 的差别是：

- reporting 只处理已经 free 的页，不强制降低 guest 可用内存。
- inflation 设定一个 balloon target，guest 会尝试 reclaim/cache eviction/swap 来交出页。
- guest physical topology 通常不变，但 balloon held pages 不能供普通 workload 使用，因此 effective available memory 下降。
- deflate 可把容量还回 guest。

bundled API 的 `PUT /balloon` 用于 pre-boot 创建设备，`PATCH /balloon` 可以在 boot 前后调整 `amount_mib`（[firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:63)）。

`deflate_on_oom=true` 是安全阀：guest 用户页分配接近 OOM 时，kernel 可从 balloon 取回一部分页。它不是主动回收机制，而且不是所有内核分配/超大分配都能触发有效 deflate；官方边界见 [ballooning.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md#L20-L38)。

传统 balloon 是 best effort，而不是可信的硬 limit：

- 达到 target 的速度由 guest driver 决定。
- target 过激会造成 guest reclaim、swap、allocation stall、OOM 和延迟尖峰。
- compromised/异常 guest driver 可以不配合。
- host 仍应准备承受 VM 使用启动时全部内存的情况。

Firecracker 官方明确将 balloon memory restriction 描述为 best effort，并建议配合 host monitoring/cgroup（[ballooning.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md#L54-L93)）。

### 5.3 Free-page hinting：host 触发的回收扫描

Firecracker 还提供 free-page hinting：host 通过 `/balloon/hinting/start` 触发 guest 扫描空闲范围，并通过 status/stop 控制。它比持续 reporting 更容易选择回收时机。

但 v1.15.1 官方文档将其标为 Developer Preview，并明确指出 guest 重用页面与 VMM discard 之间存在低概率数据损坏竞态。AgentENV 当前将 `free_page_hinting=None`，也没有 start/status/stop 调用，因此不应把它算作现有产品能力。

### 5.4 virtio-mem：真正改变 guest online memory 容量

virtio-mem 是独立于 boot memory 的 hotpluggable memory pool：

```text
boot memory: 固定基础容量，不能通过 virtio-mem 移除
hotplug pool: pre-boot 声明最大 total_size
requested size: host 运行时希望 guest plug 的目标
plugged size: guest driver 实际完成的容量
```

Firecracker v1.15.1 官方说明 `/hotplug/memory` 必须在 VM 启动前配置 maximum region，初始全部 unplug；只有这块独立 pool 能动态 plug/unplug（[memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L41-L51)）。

运行时提高 `requested_size_mib` 会异步增加 guest memory；降低 requested 会要求 guest 迁移/释放 block。guest 报告 block 已 unplug 后，Firecracker 立即释放相应 host backing；全部 block 离开一个 slot 后还会取消 KVM slot 并保护地址范围（[memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L135-L202)）。

bundled API 提供：

- `PUT /hotplug/memory`：配置 maximum hotplug pool。
- `PATCH /hotplug/memory`：改变 `requested_size_mib`。
- `GET /hotplug/memory`：读取 `plugged_size_mib` 与 requested 的收敛状态。

本地 schema 见 [firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:621)。默认 logical block 为 2 MiB，slot 为 128 MiB；状态模型见 [firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:1850)。

Firecracker 内部不是在声明 `total_size_mib` 时就给整个 maximum pool 分配同等数量的 resident physical pages。它先建立 hotplug region 的虚拟地址映射，并把各 slot 初始化为 unplugged；未 plugged slot 不注册为可访问的 KVM memory region。运行中 PATCH 只是更新 requested size 并触发 virtio config interrupt，guest driver 随后按 block 发出 plug/unplug request。处理 unplug 时，Firecracker 对对应 range 执行 `discard_range`，并根据 slot 内剩余 block 状态更新 KVM slot。因此：

```text
maximum hotplug address capacity != 启动时立即消耗同等 RSS
requested size != 已完成的 plugged size
plugged size != 必然全部 resident，未触碰页仍可按需分配
unplugged block -> discard backing；完整空 slot -> 取消/保护 KVM slot
```

对应上游实现可参考 [builder.rs：建立 hotplug region](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/src/vmm/src/builder.rs#L170-L188)、[memory.rs：初始 unplugged slots](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/src/vmm/src/vstate/memory.rs#L251-L265) 和 [virtio-mem device：unplug/discard 与 requested size](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/src/vmm/src/devices/virtio/mem/device.rs#L529-L596)。

virtio-mem 的重要限制：

- x86_64 guest kernel 至少 5.16，aarch64 至少 5.18。
- guest 必须启用 `CONFIG_VIRTIO_MEM`。
- 推荐 `memhp_default_state=online_movable` 才更容易可靠 hot-remove。
- 被 pin、不可迁移或落入不可移动 zone 的页会让 shrink 长时间不能达到 requested target。
- PATCH 成功只说明接受目标，不代表 guest 已经完成，必须轮询 `plugged_size_mib`。
- hotplug pool 的 `struct page` metadata 可能消耗基础 boot memory。
- Firecracker 的 block discard 失败会记录指标/日志，但不是对 guest 暴露的硬失败，因此仍应以 host RSS/cgroup usage 验证实际释放量。

前置条件和 movable zone 说明见 [memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L21-L39) 与 [memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L204-L249)。

unplugged block 的 backing 通过类似 `MADV_DONTNEED` 的方式按 block 返回宿主；安全与 guest cooperation 边界见 [memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L265-L300)。

## 6. AgentENV 当前到底接入了什么

### 6.1 已接入：`amount=0 + free_page_reporting=true`

fresh boot 配置 balloon 的源码是：

```rust
Balloon {
    amount_mib: 0,
    deflate_on_oom: true,
    stats_polling_interval_s: None,
    free_page_hinting: None,
    free_page_reporting: Some(true),
}
```

实现见 [instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:387)，调用点见 [sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:1617)。

这意味着：

- traditional balloon target 是 0，没有主动 inflation。
- `deflate_on_oom` 当前基本没有已 inflated pages 可供 deflate，只是未来/外部修改 target 时的安全网。
- free-page reporting 持续工作，能够在线降低 free pages 对应的 host RSS。
- free-page hinting 未启用。
- VM 不需要 pause，进程和 workload 保持运行。

resume 路径不会重新创建 balloon，因为设备 topology 和 state 已在 `vm_state.bin` 中恢复（[sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:1265)）。这也意味着旧 snapshot 若创建时没有 balloon，恢复后依然没有 free-page reporting；snapshot load 后不能补装一个新的 balloon device。

### 6.2 DAMON + reporting 的组合

AgentENV 默认 guest boot args 启用 `damon_reclaim`，包括 60 秒冷页年龄、每轮 quota 和 `skip_anon=Y`（[default.toml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/config/default.toml:15)）。设计意图在源码注释中很清楚：

```text
DAMON 回收 guest 中的冷 page cache
  -> 页面进入 guest free/buddy allocator
  -> free-page reporting 报告给 Firecracker
  -> Firecracker discard host backing
  -> host RSS 下降
```

见 [sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:1617)。

因为 `skip_anon=Y`，这套默认策略的目标是冷 file-backed/page-cache，而不是仍被应用持有的匿名 heap。以下内存不会因为这套组合自动消失：

- agent process 尚未 free 的 heap。
- pinned pages。
- mlocked memory。
- 活跃 dirty anonymous working set。
- guest kernel 不能回收或迁移的页。

另外，仓库只记录 bundled kernel 为 `vmlinux-6.1.175`（[deps_manifest.toml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/config/deps_manifest.toml:5)，没有发布对应 kernel `.config` 或可复现源码。生产部署需要实际验证 `CONFIG_VIRTIO_BALLOON`、`CONFIG_PAGE_REPORTING`、DAMON_RECLAIM 及相关参数是否都由该二进制支持，而不能只根据 boot args 推断已生效。

### 6.3 未接入：dynamic balloon target 和 statistics

虽然 Firecracker 支持运行时 `PATCH /balloon`，AgentENV 的 `FirecrackerInstance` 没有对应 wrapper，外部 OpenAPI 也没有 memory target/resize endpoint。

更关键的是当前 `stats_polling_interval_s=None`，即 pre-boot 没有启用 balloon statistics。Firecracker schema 明确规定 statistics 不能在 boot 后从 disabled 切成 enabled；运行中只能调整已启用的 polling interval（[firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:1071)）。

所以当前 AgentENV 缺少安全 balloon controller 所需的信号：

- target/actual balloon size。
- guest free/available memory。
- swap in/out。
- major/minor faults。
- OOM kill、allocation stall、reclaim scan/reclaim amount。
- per-sandbox Firecracker RSS/PSS。

在没有这些反馈时，直接按 host pressure 大幅提高 balloon target 很容易把节点压力转化成 guest swap、OOM 和尾延迟。

### 6.4 未接入：virtio-mem

generated Firecracker model 已包含 `MemoryHotplugConfig`、size update 和 status，但 AgentENV runtime 没有调用 `/hotplug/memory`：

- launch config 没有 base/max/requested hotplug fields。
- `FirecrackerInstance` 没有 PUT/PATCH/GET wrapper。
- AgentENV OpenAPI 没有 sandbox resource resize route。
- guest boot args 没有 `memhp_default_state=online_movable`。
- orchestrator/scheduler 没有 requested/plugged/host-resident 资源模型。
- 自定义 OverlayBD memory snapshot path 尚无 virtio-mem 多 region/holes 的端到端测试证据。

Firecracker 官方说明 snapshot 会把 unplugged areas 表示为 sparse holes（[memory-hotplug.md](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md#L302-L311)），但这只能证明 Firecracker 本身的格式语义，不能直接推定 AgentENV 的 direct dirty-range -> OverlayBD -> ublk restore 链路已经兼容。

## 7. “部分释放”能释放多少：一个示例

假设一个 VM 配置 4 GiB boot memory，当前内部状态为：

```text
1.5 GiB active anonymous working set
1.0 GiB file/page cache
1.5 GiB free pages
```

不同机制的理想化结果：

| 机制 | 可能回收的内容 | guest 感知 | host 结果 |
|---|---|---|---|
| 当前 free-page reporting | 1.5 GiB free pages 的大部分 | MemTotal 仍约 4 GiB | RSS 可下降约对应 backing，之后可回升 |
| DAMON + reporting | 再将部分冷 1 GiB cache 回收成 free 后报告 | cache miss 可能增加 | RSS 进一步下降 |
| balloon target=2 GiB | guest 尝试交出约 2 GiB pages | effective available memory 明显降低 | RSS 下降，但可能触发 reclaim/swap |
| virtio-mem | 仅能缩预配置 hotplug pool，不能拿走 4 GiB boot base | MemTotal/online memory 下降 | unplug block backing 释放 |
| pause+snapshot+stop | active、cache、free 对应的整个 VM runtime | VM 停止服务 | 私有 RSS 最彻底释放，状态落盘 |

这是语义示例，不是精确 RSS 预测。真实值还受 Firecracker/VMM 元数据、页表、共享 page cache、dirty/COW pages、THP、guest cooperation 和内核回收效率影响。

若希望 virtio-mem 真正实现 1～4 GiB 弹性，设计应是：

```text
boot memory = 1 GiB（不可缩基础）
virtio-mem total hotplug pool = 3 GiB
requested = 0～3 GiB
guest online total = 约 1～4 GiB
```

不能先创建 4 GiB boot memory，再期望 virtio-mem 把 boot base 缩成 1 GiB。

## 8. 资源计量为什么看不到部分回收收益

Orchestrator 的 `allocated_memory_bytes` 始终按：

```rust
resources.memory_mib * 1024 * 1024
```

计算。只有 sandbox 进入 `Paused`，active allocation 才归零并转入 paused reservation（[metrics.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/orchestrator/metrics.rs:87)）。

因此，即使 free-page reporting 已把一个 4-GiB VM 的 Firecracker RSS 降到接近 1 GiB：

- AgentENV 仍上报 allocated memory = 4 GiB。
- scheduler 的 allocated-percent 阈值不会因此释放 3 GiB placement capacity。
- 只可能通过 node-level `memory_used_bytes` 间接看到宿主整体压力下降。
- 无法知道具体是哪一个 sandbox 回收了多少。

Node snapshot 同时输出固定 allocated 值和 host aggregate usage（[service.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/observability/service.rs:69)）。当前没有 per-sandbox Firecracker cgroup `memory.current`、RSS/PSS 或 balloon actual/reclaimed metrics。

所以 AgentENV 当前 memory ballooning 是：

> 物理 RSS 的机会性优化，而不是逻辑 reservation、计费规格或调度容量的动态缩容。

## 9. 推荐的实现优先级

### P0：先把当前部分回收变得可验证、可观测

1. 发布 bundled guest kernel 的 `.config`/可复现源码，并验证：
   - `CONFIG_VIRTIO_BALLOON`
   - `CONFIG_PAGE_REPORTING`
   - DAMON_RECLAIM 和所有 boot parameters
2. 增加集成测试：VM 不 pause，制造 page cache/free pages 后验证 Firecracker RSS/PSS 下降，同时 guest `MemTotal` 不变。
3. 验证 pause/resume 后 free-page reporting 仍继续工作。
4. pre-boot 打开 balloon stats polling；旧 snapshot 增加 capability/version 标记并逐步重建。
5. 为每个 Firecracker runtime 建立 cgroup，采集 `memory.current`、`memory.stat` 和 CPU usage。
6. 指标拆分：
   - configured memory
   - guest available memory
   - balloon target/actual
   - virtio-mem requested/plugged
   - host resident memory

### P1：实现受控 traditional balloon

新增内部 `PATCH balloon target`，但必须采用渐进闭环：

```text
观察 host pressure + guest available + swap/fault/OOM + latency
  -> 小步 inflate
  -> 等待 actual 收敛
  -> 压力/延迟越界立即停止或 deflate
```

不能把 balloon target 当成可信的硬 memory limit；真正的宿主保护仍需 per-sandbox cgroup `memory.high/max` 和 node admission。

### P2：接入 virtio-mem 做真正容量弹性

需要同时修改：

- public API：base/max/requested memory 或 `PATCH /sandboxes/{id}/resources`。
- Firecracker wrapper：PUT/PATCH/GET `/hotplug/memory`。
- guest kernel/cmdline：virtio-mem、memory hotplug、movable zone。
- metadata：base、max、requested、plugged、host resident。
- scheduler：扩容前原子预留，缩容完成后才释放 reservation。
- snapshot：保存 capability/version，测试 unplugged holes、direct OverlayBD、ublk restore。
- failure semantics：PATCH accepted 但 guest 不收敛时的 timeout/rollback。

### P1：CPU 先做 quota elasticity，而不是等待 hotplug

CPU 方面建议优先：

1. guest 预配 peak vCPU topology。
2. per-sandbox cgroup `cpu.max`/`cpu.weight`。
3. 暴露当前 CPU entitlement，而不是把 topology 当作唯一 allocation。
4. scheduler 在放宽 entitlement 前原子检查 headroom。
5. workload burst detector 使用 run queue、CPU throttling 和 latency，而不是只看瞬时 CPU percent。

这条路径可以在不修改 Firecracker CPU topology和 snapshot schema 的情况下解决大部分“1 核常态、4 核 burst”的需求。

## 10. 最终判断

### 没有 vCPU hotplug 的影响

它迫使 AgentENV 在“burst 性能”和“逻辑调度密度”之间静态选边：低配导致 VM 内排队和尾延迟，峰值预配导致 reservation 浪费，overcommit 又会在同步 burst 时转化为宿主争用。对长生命周期、强状态、阶段性并行的 Agent sandbox，这个问题比对短小无状态请求更突出。

不过，解决这一问题不必首先等待真正 CPU hotplug。固定 peak topology 加动态 cgroup quota 通常更容易实现，也更兼容 Firecracker snapshot。

### Firecracker 是否支持运行态部分释放内存

支持，而且 AgentENV 已使用其中最温和的一种：

- **当前 AgentENV：** free-page reporting，在线丢弃 guest free pages 的 host backing；guest 容量不变。
- **Firecracker 已支持、AgentENV 未控制：** traditional balloon target，主动降低 guest effective available memory。
- **Firecracker 已支持、AgentENV 未接入：** virtio-mem，真正动态调整 boot base 之外 hotplug pool 的 online capacity。
- **AgentENV Pause：** snapshot 后终止 VMM，释放最彻底，但不是运行态部分回收。

因此不能笼统说“AgentENV 已支持动态内存扩缩容”。准确说法应是：

> AgentENV 已支持运行态空闲页 backing 的部分回收；尚不支持用户/调度器可控的 balloon target，也不支持 virtio-mem 意义上的 guest memory capacity resize。

## 11. 参考资料与源码索引

### AgentENV 本地源码

- Firecracker 版本：[config/deps_manifest.toml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/config/deps_manifest.toml:1)
- cold resource API：[src/api/impls/sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/api/impls/sandbox.rs:252)
- fresh CPU/memory config：[src/sandbox/firecracker/factory.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/factory.rs:100)
- `/machine-config` wrapper：[src/sandbox/firecracker/instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:275)
- AgentENV balloon config：[src/sandbox/firecracker/instance.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/instance.rs:387)
- fresh/resume balloon lifecycle：[src/sandbox/firecracker/sandbox.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/sandbox.rs:1265)
- DAMON boot args：[src/sandbox/firecracker/config.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/sandbox/firecracker/config.rs:27)
- snapshot 不允许改 CPU/内存：[src/template/builder.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/template/builder.rs:234)
- 固定资源计量：[src/orchestrator/metrics.rs](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/src/orchestrator/metrics.rs:56)
- Firecracker balloon API schema：[thirdparty/firecracker-client/firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:63)
- Firecracker virtio-mem API schema：[thirdparty/firecracker-client/firecracker.yaml](/mnt/u/hukeyang/AKernel/kvcache-ai/AgentENV/thirdparty/firecracker-client/firecracker.yaml:621)

### Firecracker v1.15.1 官方资料

- [Using the balloon device with Firecracker](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md)
- [Virtio-balloon free-page reporting](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/ballooning.md#L307-L347)
- [Memory Hotplugging with virtio-mem](https://github.com/firecracker-microvm/firecracker/blob/v1.15.1/docs/memory-hotplug.md)
- [Linux memory hotplug documentation](https://docs.kernel.org/admin-guide/mm/memory-hotplug.html)
