# AKernel sandboxd gVisor(runsc)Checkpoint/Restore 现状 Survey —— 兼 AFaaS "seed" 设计的对位分析

> 调研时间:2026-08-24(Asia/Shanghai);文档时间戳 `20260824T151500Z`
> 缘起:AKernel_CR 开工前,先盘清**现有 gVisor runtime 的 C/R 是怎么做的**、论文([Chai et al., OSDI'25 "Fork in the Road"](../../References/Chai et al_2025_Fork in the Road.pdf),AFaaS/Ant Group)的 seed 设计与它的对位关系,以及哪些能力已就绪但**尚未接线**。
> 代码证据(2026-08-24 逐行核对):
> - sandboxd:`/home/keyang/AKernelWorkspace/AKernel_CR/sandboxd`(main,HEAD `d98d8ae`),核心在 `pkg/runtime/runsc/`;
> - gVisor fork:`/home/keyang/AKernelWorkspace/gvisor`(akernel-dev,分支 `scheduler-adopt-filestore`,`release-20260817.0-51-g1985ab80b`,含 PR#14228 全部工作);
> - pinned 运行时:`sandboxd/third_party/runtime-versions.env`(runsc = `release-20260817.0-akernel.1`)。
> 姊妹篇:[PR#14228 优化评估](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md)、[YuanRong fork 启动调研](20260812T163000Z-yuanrong-fork-launch-akernel-sandbox-path-survey.md)、[FC CR 实施计划](../../AKernel_CR/Docs/plans/20260824T-cr-fc-template-incremental-ondemand-plan.md)

---

## 1. 结论摘要

1. **现状是一条纯上游语义的全量 C/R**:Checkpoint 走 `runsc checkpoint` CLI(单命令),Restore 不走 CLI 而走 runsc 控制 socket 的 urpc RPC(`containerManager.Restore`,checkpoint 文件以 FD 直传)。可写层内容**整体序列化进 image**,O(S),无增量、无共享。
2. **一个隐藏的正确性陷阱**:`compress=false`(proto3 bool 默认值!)时 `runsc checkpoint --compression=none` 会把 image 拆成 `checkpoint.img + pages.img + pages_meta.img` 三个文件,而 sandboxd 的 restore 只打开 `checkpoint.img` 单文件、不置 `HavePagesFile` —— **compress=false + runsc 目前是一个会静默丢内存的组合**;e2e 默认 `compress=true` 单页文件格式恰好绕开。没有代码层防御。
3. **三块"已就绪未接线"的能力**:(a) restoreOpts 里的 `Background` / `UseCheckpointGofer`(异步/远程 checkpoint 流式服务,上游为 CUDA checkpoint 场景建的前缀)声明了但从不设置;(b) **PR#14228**(可写层外部化 + FICLONE + clone-on-adopt,fanout 103ms/份、磁盘 +0)在本地 gVisor fork 分支上完整存在,但 pinned 运行时不含、sandboxd 一个 flag 都没传;(c) 上游"同一 image 恢复 N 份"的天然多路复用没有任何管理面(seed/template 概念在 sandboxd 中不存在)。
4. **论文的 seed 设计 ≠ gVisor 的任何现成功能**:AFaaS 的 seed 是"含初始化态(语言运行时/依赖/用户代码)的沙箱母本 + 树状层级 + CoW 多级 fork"(clone() 路线,Catalyzer sfork 血统,冷启动 5.45–9.41ms)。gVisor 侧**最接近的数据面等价物**是 PR#14228 的"(1MB state image + filestore FICLONE 快照) 对 → clone-on-adopt → N 个写隔离沙箱";但论文的**树状 seed 池、多级 fork、控制面 Register/派生协议**(YuanRong `make_seed`/`template_id` 落地形态)在 AKernel 栈里完全缺位——这正好是 AKernel_CR FC 计划 M3(模板)要补的同一层。
5. 对 FC CR 计划的意义:gVisor 这条线的经验(压缩格式耦合、filestore 外部化、fanout 实测数字、restore 不走 CLI 走 RPC 的理由)直接约束 FC 侧的 artifact 设计与里程碑验收线。

---

## 2. 完整调用链(现状,逐行)

### 2.1 Checkpoint:`runsc checkpoint` CLI

`pkg/runtime/runsc/handler.go:183` → `client.go:256-285`:

```sh
runsc --root <root> [--platform=<p>] [--ignore-cgroups] \
      checkpoint --compression=none|flate-best-speed \
      [--leave-running] --image-path <checkpoint_dir> <sandbox_id>
```

- `config.Compress=false → "none"`,true → `"flate-best-speed"`(client.go:267-271);
- `leave_running` 直接映射 CLI flag;源沙箱由 runsc 自己 pause → dump → (可选)resume;
- 产物落在 caller 拥有的 `checkpoint_dir`,契约上对 sandboxd 不透明(`doc/checkpoint-restore.md`)。

### 2.2 Restore:不走 CLI,走控制 socket urpc

`handler.go:193-278` → `client.go:306-365`,四步:

1. **重建沙箱外壳**:与正常 Start 完全同路 —— OCI bundle 生成(EROFS rootfs + `RootFileOverlay(filestoreDir, size)` 可写层挂进 runsc overlay 配置,handler.go:279-288)→ `runsc Create`(新 sandbox 进程、新控制 socket);
2. **装网络**:`OpenTAP` 打开新 TAP → `contMgrSetNetworkArgs` RPC(30s 超时)把 IP/MAC/网关装进新沙箱(对应 FC 侧 `network_overrides` 的角色);
3. **传工件**:`os.Open(checkpoint_dir/checkpoint.img)` 单文件;若 `platform=kvm`,再打开 `/dev/kvm` 设备 FD 一并放进 payload(client.go:344-358);
4. **`containerManager.Restore` RPC**(`runsc/boot/controller.go:576+`):要求容器处于 `created` 态,sentry 内核在原进程内被销毁重建;成功后 `markStateRunning`。

**为什么不走 CLI**:`runsc restore` CLI 会自己起新沙箱进程,而 sandboxd 需要在 restore **之前**完成 bundle/网络/cgroup 资源编排(restore 是 Start 的一种,契约如此)——所以用"先 Create 空壳、再 RPC 注入状态"的次序。FC 侧没有这个约束(restore 就是新 FC 进程 + API overrides),两条路线形不同、神相同。

### 2.3 artifact 格式与 compress 陷阱(重点)

gVisor 的 image 目录格式由 `pkg/sentry/state/checkpointfiles/checkpointfiles.go:20-34` 定义:

| 压缩 | 目录内容 | restore payload |
|---|---|---|
| **flate(压缩)** | **单文件 `checkpoint.img`**(state + pages 内嵌) | `[checkpoint.img]`,`HavePagesFile=false` |
| **none(不压缩)** | `checkpoint.img`(state)+ `pages.img`(页内容)+ `pages_meta.img`(页元数据)三文件 | 需 `[state, pagesMeta, pages]` 三 FD,`HavePagesFile=true`(controller.go:706-744 `getRestoreReadersForLocalCheckpointFiles` 按 FD 序列取) |

sandboxd 的 restore(client.go:359-363)**永远只传单文件、永远不置 `HavePagesFile`** —— 这只在压缩单文件格式下正确。而 `CheckpointRequest.Compress` 是 proto3 bool,**默认 false**;`internal/server/checkpoint.go:106` 原样透传,无校验。即:任何调用者默认参数跑 runsc checkpoint,都会产出三文件布局,restore 却只喂 state 文件 —— **行为不是报错而是缺页恢复**(getRestoreReaders 在 `!HavePagesFile` 时直接返回 stateFile,controller.go:721-724)。e2e 工具 `test/e2e/checkpoint-restore/main.go:68` 默认 `compress=true`,说明作者知道,但契约文档 `doc/checkpoint-restore.md` 只说"runtime-specific encoding",没写这个硬约束。**建议修复:runsc Checkpoint 强制压缩,或 restore 侧检测三文件布局并补 FD**。

### 2.4 可写层与网络/设备的恢复语义

- **可写层(filestore)**:runsc 的 writable layer 本来就是宿主文件(`--overlay2=root:dir=<filestoreDir>,size=<N>`,client.go:140 + 166-172),**不进 checkpoint image 的是它的 FD 生命周期,进 image 的是它的全部内容**(O(S) 序列化,PR#14228 改的就是这一点);
- **网络**:TAP 每次 restore 重开、IP/MAC 由 SetNetworkArgs 重装 —— 网络身份是"新分配"而非"恢复"(与 FC 的 `network_overrides` 同型;gVisor dind survey 已证明 sandbox 内自组网设备不恢复是两条 runtime 共同边界);
- **KVM 平台**:`platform=kvm` 时 `/dev/kvm` FD 经 payload 直传(controller.go `HaveDeviceFile`),restore 不需要重新 openat 路径(seccomp 友好)。

---

## 3. 三块"已就绪、未接线"

### 3.1 restoreOpts 的三个哑字段(client.go:287-295)

| 字段 | 上游语义 | sandboxd 现状 |
|---|---|---|
| `HavePagesFile` | payload 含 pages_meta/pages 三 FD(§2.3) | 从不置位(= 只支持单文件压缩格式) |
| `Background` | 后台恢复(不等 restore 完成即返回) | 从不置位 |
| `UseCheckpointGofer` | 第一个 FD 不是文件而是 **UDS,连到一个实现 `stateipc.AsyncFileServer` 的外部服务进程**,checkpoint 内容经它异步/远程流式供给(gVisor 为 CUDA checkpoint / 异步落盘建的前缀,`pkg/sentry/control/state.go:130-133`、`runsc/boot/controller.go:700`) | 声明了 json tag,从不置位 |

`UseCheckpointGofer` 是 gVisor 生态里最接近"外部内存供给服务"的钩子 —— 与 FC 侧 uffd 后端、AgentEnv uffd-core 同族(且同样是**只管供给、不管记账**的定位)。

### 3.2 PR#14228:数据面原语完整存在,零接线

本地 gVisor fork 分支 `scheduler-adopt-filestore`(`release-20260817.0-51-g1985ab80b`,HEAD 三连:`in-window filestore snapshots with runsc-issued artifact manifest` → `verified artifact adoption with clone-on-adopt` → `permit FICLONE in the sandbox seccomp filter`)提供:

- `checkpoint --skip-filestore-pages`:可写层只存段元数据(ContentExternal),state image 退化为 ~1MB 纯内存态模板;
- `--filestore-snapshot-dir`:冻结窗口内 sentry 自己 FICLONE 快照可写层 + `filestores.json` 指纹清单;
- `restore --filestore-adopt-dir [--filestore-clone-on-adopt]`:指纹校验领养 + 再 FICLONE 一份私有 CoW → **一对工件恢复出 N 个写隔离沙箱**(实测 fanout 102–113ms/份,XFS df ≈ 0,见 041720Z survey)。

但:(a) `runtime-versions.env` pin 的 `release-20260817.0-akernel.1` 按 sandboxd/CLAUDE.md 自述只带 TAP readv 与 KVM 地址宽修正,**不含这批 commit**;(b) sandboxd 全仓 grep `skip-filestore|filestore-snapshot|filestore-adopt` **零命中**。即:数据面已到"可评估"成熟度(041720Z 的 parkd 原型验证过),管理面一行都没接。

### 3.3 多路复用与 seed:上游天然支持,管理面空白

上游 runsc 的 restore 对 image 文件**只读不写**,同一 `checkpoint_dir` 天然可以被 N 次 Start 恢复(内存页只读共享语义取决于 pages 文件的 mmap 方式);PR#14228 的 clone-on-adopt 更是把磁盘侧也变成 FICLONE 共享。但 sandboxd 的契约是"caller owns artifact":**没有 template/seed 注册、没有池化、没有兼容性元组、没有派生协议** —— 这一层在 YuanRong 的设计里是 `Register` RPC + `make_seed` + `template_id`(20260812T163000Z survey),在 AFaaS 论文里是完整的一章(§4.3)。

---

## 4. AFaaS "seed" 设计与 gVisor/AKernel 栈的对位

### 4.1 论文里的 seed 是什么(OSDI'25 §4.2–4.3)

- **定义**:"seed(sandbox template, a sandbox contains the initialized state to skip the initialization)"—— 把**可继承、可共享的资源**(语言运行时、已加载依赖、JIT 结果、预编译 seccomp 规则)整备进一个沙箱母本;
- **扩展到用户代码**:把高频/耗时例程预执行进 seed;
- **树状 seed + CoW**:seed 组织成层级树,子 seed 复用父 seed 内存,新实例 fork 出来后做完增量任务(依赖导入、JIT)再 pause 成为新 seed —— **增量按 CoW 记账,树按需生长**;
- **多级 fork**:请求到达时高层运行时按层级搜索选 seed → `clone()` 派生(Catalyzer sfork 血统);
- **数字**:端到端冷启动 5.45–9.41ms(重负载 6.97–14.55ms),比 Catalyzer 快 1.80–8.14×;产线 18 个月。
- 论文的三个盲区论断同样值得抄:控制路径延迟、高并发资源争抢、用户代码初始化——**冷启动不是单一组件的事**。

### 4.2 对位表:论文 seed ↔ gVisor 数据面 ↔ AKernel 现状 ↔ FC 计划

| AFaaS seed 构件 | gVisor/AKernel 侧等价物 | 现状 | FC CR 计划对应(M 序号) |
|---|---|---|---|
| seed 母本(初始化态沙箱) | 预热 checkpoint(state image + filestore 快照对) | 可手工达成,无管理 | M3 模板(三元组 + 兼容元组) |
| clone() 派生 | restore-from-image / **PR#14228 clone-on-adopt**(102–113ms/份,磁盘 0) | 数据面就绪未接线 | M3 spawn(FICLONE overlay + MAP_PRIVATE mmap) |
| 树状 seed + CoW 增量 | 无(单代快照链) | **空白** | M2 增量循环(base 代际链)是最接近的类比 |
| 多级 fork(派生后再当 seed) | 无(adopt 后的沙箱再 checkpoint 未验证指纹链) | 空白 | 未规划(可后补) |
| Register/template_id 控制面 | YuanRong `make_seed` 协议(未开源);sandboxd 无 | 空白 | M3 模板 CLI/注册表(放 AKernel 上层) |
| 用户代码预载 | 预热钩子(自由) | 无 | M3 预热钩子 |
| fork 路径的延迟(论文 §4.1 控制路径) | sandboxd Start 编排本身 | 未测量 | M0 基线任务 |

**结论**:论文 seed 的**数据面语义**(一次整备、N 次 CoW 派生、增量成长)在 gVisor 栈里靠 PR#14228 已经可表达;**缺的是控制面**(注册/池/层级/兼容)与**增量语义**(树状 CoW 只记 delta)。这两块恰好也是 FC CR 计划 M2/M3 的主体——两条 runtime 线最终共享同一个上层模板管理设计,gVisor 先行打通数据面(PR 已备),FC 先行打通增量记账(pagemap/soft-dirty),合流点在上层。

---

## 5. 给 AKernel_CR FC 计划的直接输入

1. **artifact 单/多文件耦合是前车之鉴**:FC 计划的 v2 manifest 布局必须把"格式由字段显式声明"作为硬规则(runsc 线的 compress 陷阱就是隐式耦合的代价);
2. **restore 走 RPC 而非 CLI 的理由**(编排先于注入)在 FC 侧同样成立:sandboxd FC restore 已经是"新进程 + API overrides",与 runsc 的 Create-then-Restore 次序对齐,`checkpoint_handler.go` 两条 runtime 的清理/失败语义已统一,无需改;
3. **fanout 验收线**:PR#14228 实测 103ms/份、磁盘 +0 —— FC 计划 M3 的 spawn 验收(目标 <300ms/实例)应以此为同机对照,而非只对照 AgentEnv fork 的 195–250ms(那是 O(dirty) 物化的);
4. **seed 控制面两条线共建**:Register/template_id/兼容元组/池化不应绑死 runtime,gVisor(状态对: image+filestore 快照)与 FC(三元组)只是 artifact 形状不同;
5. **`UseCheckpointGofer` 与 uffd 同族**:两者都是"外部内存供给"钩子,都不可用于记账 —— 计划里 uffd 的红线(M4)在 gVisor 线同样适用。

## 6. 建议的后续动作(按优先级)

1. **修 compress 陷阱**(sandboxd 小改动):runsc Checkpoint 强制 `flate-best-speed` 或 restore 侧三文件布局检测 —— 这是在任何严肃使用前的正确性前置;
2. 把 PR#14228 分支 pin 成新 `release-…-akernel.N` 并在 sandboxd 传 flag(e2e 先行),让 runsc 线拿到 O(1) restore + fanout 能力(评估数据已有,风险低);
3. 设计 runtime 无关的模板/seed 控制面(YuanRong Register 协议 + AFaaS 树状 seed 的简化版),作为 FC M3 与 gVisor 线的共享上层;
4. 树状 seed/多级 fork(fork 出的沙箱再作为 seed)作为长期项评估——PR#14228 的指纹链与 FC 的代际 base 都还没有"无限层级"的验证。

---

## 7. 证据索引

### sandboxd(`AKernel_CR/sandboxd`,main @ `d98d8ae`)

- runsc Checkpoint CLI 封装:`pkg/runtime/runsc/client.go:256-285`(compression 映射 :267-271、leave-running :272-274)
- Restore RPC 路径:`client.go:306-365`(OpenTAP→SetNetworkArgs→open image→payload(+KVM FD :344-358)→contMgrRestore);`handler.go:193-278`(Create 空壳→Restore 次序、失败 cleanup)
- restoreOpts 三哑字段:`client.go:287-295`(HavePagesFile/Background/UseCheckpointGofer,全仓无置位点)
- 可写层:`handler.go:279-288` `resolveRootOverlay` → `RootFileOverlay(filestoreDir, size)`(client.go:166-172 生成 `root:dir=<dir>,size=<N>`,:140 拼进 `--overlay2=`)
- compress 透传无校验:`internal/server/checkpoint.go:106`;proto 默认:`api/runtime/v1/sandbox-api.pb.go:1536-1537`;e2e 默认 true:`test/e2e/checkpoint-restore/main.go:68`
- 契约:`doc/checkpoint-restore.md`(restore=Start+checkpoint_info;caller owns artifact;增量/迁移 out of scope)
- pinned 运行时:`third_party/runtime-versions.env:20`(runsc `release-20260817.0-akernel.1`);发布物自述:`sandboxd/CLAUDE.md`(仅 TAP readv + KVM 地址宽修正)

### gVisor fork(`AKernelWorkspace/gvisor`,分支 `scheduler-adopt-filestore` @ `1985ab80b`)

- image 文件名与布局:`pkg/sentry/state/checkpointfiles/checkpointfiles.go:20-34`(checkpoint.img/pages.img/pages_meta.img)
- RestoreOpts 与 FD 序语义:`runsc/boot/controller.go:555-576`(payload 注释)、`:699-744` `getRestoreReaders`(:721-724 单文件分支 = compress 陷阱的机制点)
- checkpoint gofer:`pkg/sentry/control/state.go:120-133`(AsyncFileServer over UDS)、`runsc/boot/controller.go:745+`
- PR#14228 flag 面:`runsc/cmd/checkpoint.go:76-79`(--skip-filestore-pages/--filestore-snapshot-dir 及其约束)、adopt 见 HEAD 三连 commit;seccomp FICLONE 放行:`1985ab80b`

### 论文与其他

- AFaaS seed 设计:`References/Chai et al_2025_Fork in the Road.pdf`(§1 摘要、"Tree-structured Seeds"/"Multi-Level Fork" §4.3、5.45–9.41ms 评估;本地全文 `/tmp/fork-road.txt`)
- PR#14228 实测(park/restore/fanout 数字):[20260821T041720Z](20260821T041720Z-pr14228-runsc-cr-optimization-eval-vs-agentenv-substrate.md) §1/§3
- YuanRong seed 控制面(Register/make_seed/template_id):[20260812T163000Z](20260812T163000Z-yuanrong-fork-launch-akernel-sandbox-path-survey.md) §4
- gVisor C/R 边界(dind、网络不恢复):[20260820T082000Z](20260820T082000Z-gvisor-dind-checkpoint-restore-survey.md)
- FC 侧对位计划:`AKernel_CR/Docs/plans/20260824T-cr-fc-template-incremental-ondemand-plan.md`
