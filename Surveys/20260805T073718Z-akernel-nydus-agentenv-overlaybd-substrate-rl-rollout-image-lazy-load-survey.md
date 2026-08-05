# AKernel Nydus、AgentENV OverlayBD 与 Agent Substrate 镜像路径：源码机制、性能证据及 RL Rollout 并发分析

> 调研时间：2026-08-05 15:37:18（Asia/Shanghai）
>
> 文档时间戳：`20260805T073718Z`
>
> AKernel commit：`5684c858cd835db053adef1265419af64e67bd04`
>
> AKernel `distill-fs` submodule commit：`5f3d6d3979a4ed76d4bb6722d7c1ce6db7126634`
>
> AKernel `sandboxd` submodule commit：`60e6a33e1f2b1ae04c10586c238f6d0200d0c315`
>
> AgentENV commit：`6296bc4be7ad79eb3a278eb5264ef011c341adf5`
>
> Agent Substrate commit：`cbdeb7dbe003a55a16960a301bc595d9aa38b1ad`
>
> 方法：固定版本源码、项目文档、DADI、FaaSNet 与 AFaaS（*Fork in the Road*）论文交叉核对。本文没有在同一台服务器上完成三系统的 head-to-head benchmark；所有性能结论均标注为“论文实测”“项目文档观察”或“源码机制推断”。

## 1. 结论摘要

### 1.1 最重要的六个结论

1. **AKernel 不是直接运行 stock `nydusd`。** 它兼容标准 Nydus OCI/RAFS 格式并固定依赖 Nydus Rust crates，但实际 data path 是 AKernel 自己的 `distill_fs`：一个只读 FUSE 文件系统，当前只支持 RAFS v5。更准确的说法是“基于 Nydus RAFS 格式和 Nydus 库的自定义 lazy filesystem”。[`distill-fs/Cargo.toml:29-32`](../../akernel/src/distill-fs/Cargo.toml#L29) [`image/nydus.rs:169-195`](../../akernel/src/distill-fs/src/image/nydus.rs#L169)

2. **AgentENV 只有 OverlayBD-native 镜像才是真正的 runtime lazy load。** Native 镜像只生成 `repoBlobUrl + digest + size` 的远端 lower 描述，运行时由 `registryfs_v2` 发 HTTP Range 请求；普通 OCI tar 镜像在 converted commit cache 未全命中时，仍要执行完整 `regctl image copy` 并逐层转换成 OverlayBD `.commit`，完成后才能返回。因此“AgentENV 支持 lazy image”不能泛化为任意 `userImage`。[`oci_image.rs:198-275`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L198) [`oci_image.rs:338-438`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L338)

3. **二者都把“先下载整个镜像”改成了“先读小元数据、首访再取数据”，但抽象层不同。** AKernel 是“路径名/inode → RAFS file chunk → blob range”的文件级路径；AgentENV 是“guest ext4 block → ublk → LSMT logical block mapping → ZFile encoded block → registry range”的块级路径。前者更自然地做文件语义和跨镜像 chunk 去重；后者更适合 Firecracker 的 virtio-blk、可写 block upper、rootfs/drive/memory snapshot 的统一表示。

4. **当前 Agent Substrate 没有同类 lazy rootfs。** 它在冷节点上完整 download、decompress、untar OCI layers，然后把 node-local unpacked layer pool 作为 overlayfs lowerdir。它非常擅长 warm-node 复用，但 all-cold-node burst 仍有每节点完整拉取/解包成本；lazy backend、cache-aware scheduling 和 `PreloadImage` 仍在文档所述的后续阶段。[`imagecache/README.md:74-103`](../../agent-substrate/substrate/internal/imagecache/README.md#L74) [`imagecache/README.md:158-178`](../../agent-substrate/substrate/internal/imagecache/README.md#L158)

5. **没有当前源码支持的直接性能排名。** DADI 论文证明了 OverlayBD/DADI 这类块级 lazy + prefetch + P2P 在其环境中的可扩展性；FaaSNet 又证明 1→N burst 下，按块 lazy 仍可能被中心 registry 或 P2P root 拖慢，拓扑化流式传播可以进一步提升。但这些数字不能直接归属于当前 AgentENV、AKernel 或 Substrate。

6. **AKernel 若接入 AFaaS seed tree，会与 Substrate Golden Snapshot 形成两种不同的初始化复用路径。** 前者是节点内常驻、多层、CoW live fork，目标是把高并发实例化压到极短的本地路径；后者是对象存储中的单层 template checkpoint，目标是在释放 Worker 资源后仍可跨节点恢复。当前开源 AKernel 尚未实现 fork cold start；这是一项建议设计，不是现有能力。[`README.md:35-39`](../../akernel/README.md#L35) [`handler.go:32-43`](../../akernel/src/sandboxd/pkg/runtime/handler.go#L32)

### 1.2 条件式性能判断

| 场景 | 更可能占优 | 原因 |
|---|---|---|
| 稀疏文件访问、Python/工具依赖只触达镜像小部分、跨镜像内容重复多 | AKernel Nydus/`distill_fs` | RAFS 直接知道文件 chunk；节点级 ChunkDB 以 checksum 存放内容 |
| Firecracker microVM、ext4 rootfs、需要可写 block upper、snapshot/fork/drive 统一格式 | AgentENV OverlayBD | rootfs 原生作为 ublk block device；LSMT lower/upper 与 Firecracker 接口匹配 |
| 任意普通 OCI 镜像首次使用 | 取决于预转换 | AKernel 无 Nydus companion 时退化为完整 OCI pull/extract；AgentENV 无 native OverlayBD 时退化为完整 pull/convert |
| 同节点、同镜像、大量并发 sandbox | AKernel 当前共享边界更直接；AgentENV 也可强缓存 | AKernel 复用同一个 per-image FUSE mount；AgentENV 共享 node-local remote block/commit cache，但 rootfs ublk runtime device 仍按 sandbox 管理 |
| 多节点全部冷、P2P 默认关闭 | 二者都可能出现 origin storm | 请求总量近似节点数乘以工作集；lazy 只减少每节点字节，不自动解决 fan-out |
| 节点已完整预热、重复创建 | Substrate 可能非常快 | 无 remote first-touch，直接从 unpacked layers 组成一次 overlay mount；项目文档称 warm compose 为毫秒级 |
| warm snapshot 大量 resume | AgentENV 有镜像路径之外的额外优势 | 同一 memory snapshot 可共享只读 memory ublk/page cache；这不是 Nydus-vs-OverlayBD rootfs 的单项效果 |
| 同一语言/同一环境初始化后突发大量 rollout | AKernel + node-local seed tree 在命中时更可能有最低启动延迟；Substrate 更容易跨节点恢复 | live fork 可CoW共享父seed内存并绕过对象下载；Golden Snapshot可移植但每个Actor仍需独立download/decompress/restore |
| 执行期间顺序扫描几乎整个镜像 | lazy 优势显著下降 | 传输量接近完整镜像，额外的 FUSE/ublk、Range、解压和 cache 管理成本开始主导 |

一句话结论是：**RL rollout 的胜负不由“Nydus 还是 OverlayBD”这个名字决定，而由 image 是否预转换、节点缓存状态、实际 touched working set、同节点复用度、origin/P2P 拓扑和 first-useful-action 尾延迟共同决定。**

## 2. 范围、术语与证据边界

### 2.1 本文比较的实际对象

本文中的三个对象不是三个完全相同的组件：

- **AKernel image path**：`sandboxd image-manager + distill_fs + Nydus RAFS v5 + Nydus storage backend + runsc`；
- **AgentENV image path**：`ImageResolver + Rust OverlayBD/LSMT/ZFile + registryfs_v2 + ublk + Firecracker`；
- **Agent Substrate image path**：`atelet imagecache + node-local unpacked OCI layer pool + overlayfs + ateom gVisor/microVM`。

本文将“lazy load”限定为：sandbox 可以在镜像 data blob 尚未全部进入本地时进入 mount/boot/运行路径，后续文件或块访问按需触发远端数据读取。仅仅“缓存了已经完整解包的 layer”不算 lazy load。

### 2.2 性能证据分级

| 标签 | 含义 |
|---|---|
| 论文实测 | DADI/FaaSNet 在论文指定软硬件和 workload 下的实验结果 |
| 项目文档观察 | 仓库 README/log comment 中报告的观察，本文没有独立复现 |
| 源码事实 | 当前固定 commit 中可直接验证的执行路径、默认值或限制 |
| 机制推断 | 从源码推导出的趋势，必须通过目标环境 benchmark 验证 |

尤其需要避免以下误用：

- DADI 2020 的结果不等于 AgentENV 2026 Rust OverlayBD 实现的结果；
- FaaSNet 的实验预留了 free VM pool，**不包含 VM cold start**；
- Substrate README 的 warm-node timing 不是与 AKernel/AgentENV 同机对比；
- `spawn API 返回`、`mount ready`、`envd/readyz ready`、`first file open` 与 `first useful RL step` 是不同指标。

## 3. AKernel：Nydus RAFS + 自定义 distill_fs 的真实路径

### 3.1 端到端路径

~~~mermaid
flowchart LR
    A[Sandbox create] --> B[sandboxd MountOCI]
    B --> C{原ref或ref+_nydus_v3\n是否Nydus OCI?}
    C -->|否| D[普通OCI完整pull/extract\nOverlayFS lowerdirs]
    C -->|是| E[提取最后一层\nimage/image.boot]
    E --> F[启动/复用per-image distill_fs]
    F --> G[FUSE只读rootfs目录]
    G --> H[runsc create\nroot:memory writable overlay]

    I[应用read file] --> J[RAFS inode/chunk metadata]
    J --> K{ChunkDB hit?}
    K -->|是| L[返回chunk slice]
    K -->|否| M[nydus-storage backend\nregistry/HTTP proxy range]
    M --> N[可选per-image sparse blob cache]
    N --> O[解压/校验]
    O --> L
    O -.异步.-> P[节点级ChunkDB]
~~~

`MountOCI` 的实际分支是：先检测用户原始 ref，再尝试配置后缀；只有两者都不是 Nydus 才进入普通 OCI 路径。当前 standalone 配置后缀是 `_nydus_v3`。[`http.go:505-545`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go#L505) [`sandboxd_config.toml:45-53`](../../akernel/deploy/standalone/config/sandboxd_config.toml#L45)

这里存在两个重要边界：

- 如果已经检测为 Nydus，但 Nydus mount 失败，代码明确拒绝回退普通 OCI，避免悄悄改变格式/性能语义；
- 如果 registry 没有原 ref 的 Nydus 版本，也没有 `${ref}_nydus_v3` companion，则进入普通 OCI 完整 pull/extract，形成明显的性能悬崖。

Nydus 格式检测遵循标准 annotation：最后一层包含 `containerd.io/snapshot/nydus-bootstrap`，bootstrap 位于 `image/image.boot`。[`bootstrap.go:31-68`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go#L31) [`bootstrap.go:71-166`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go#L71)

### 3.2 为什么它不能简单称为“标准 nydusd”

`distill_fs` 固定依赖同一 Nydus git revision 的 `nydus-api`、`nydus-rafs`、`nydus-storage` 和 `nydus-utils`，说明它确实复用了标准格式、metadata parser、backend 和解压能力。[`Cargo.toml:13-32`](../../akernel/src/distill-fs/Cargo.toml#L13)

但 production path 运行的是 `distill_fs mount --src nydus`：

- `RafsSuper::load_from_file` 加载 bootstrap；
- `GeneralBackend` 从 Nydus backend JSON 创建 registry/OSS/S3/HTTP-proxy backend；
- 自己实现 `fuse_backend_rs::FileSystem` 的 lookup/read；
- 当前明确拒绝 RAFS v6，直到改为 Nydus `BlobDevice` flow；
- encrypted chunk 与 batched chunk 也不支持。

证据见 [`image/nydus.rs:169-231`](../../akernel/src/distill-fs/src/image/nydus.rs#L169)、[`image/nydus.rs:331-375`](../../akernel/src/distill-fs/src/image/nydus.rs#L331) 和 [`distill-fs/README.md:49-59`](../../akernel/src/distill-fs/README.md#L49)。因此论文或文档宜写成：

> AKernel consumes standard Nydus OCI/RAFS v5 images through a custom `distill_fs` FUSE reader built on Nydus libraries.

不宜写成：

> AKernel directly runs the standard Nydus daemon/snapshotter.

### 3.3 文件 read、缓存与去重

FUSE read 根据 RAFS superblock 的 `chunk_size` 计算覆盖区间，随后逐个 chunk 串行调用 `read_chunk_into_writer`。[`image/nydus.rs:647-684`](../../akernel/src/distill-fs/src/image/nydus.rs#L647)

单个 RAFS chunk 的处理顺序是：

1. 从 RAFS metadata 取得 blob index、compressed offset/size、uncompressed size 和 digest；
2. 先按 RAFS digest 查节点级 `ChunkDB`；
3. miss 时读取对应 compressed blob range；
4. 如有压缩则解压整个 RAFS chunk；
5. 向 FUSE 返回用户请求的 slice；
6. 完整 uncompressed chunk 通过有界队列异步写入 `ChunkDB`。

源码证据见 [`image/nydus.rs:250-375`](../../akernel/src/distill-fs/src/image/nydus.rs#L250) 与 [`image/nydus.rs:516-573`](../../akernel/src/distill-fs/src/image/nydus.rs#L516)。队列满时会丢弃 store 请求而不阻塞 foreground read，这保护了首访延迟，但会降低后续命中率。

需要区分两种 chunk：

- **RAFS file chunk**：大小由镜像 metadata 决定，是 Nydus 文件读的语义单位；
- **4 MiB backend cache/dedup chunk**：`distill_fs` 自己的 blob cache 与 raw backend 去重单位，由 `CHUNK_SIZE` 定义。[`backend/mod.rs:22-30`](../../akernel/src/distill-fs/src/backend/mod.rs#L22)

不能把“AKernel 的 Nydus chunk 固定为 4 MiB”写进论文。4 MiB 是自定义 backend cache/dedup 层，不是 RAFS 格式的统一 chunk size。

per-image sparse cache 用 payload file + bitmap 标记 4 MiB extent；同一 daemon 内相同 extent 的 concurrent miss 通过 in-flight map 合并。[`cache.rs:348-430`](../../akernel/src/distill-fs/src/backend/cache.rs#L348) 所有 image daemon 又共享同一个 node-level LMDB `ChunkDB`，从而可能跨 image 复用相同 checksum 内容。

### 3.4 节点内复用、首次屏障与写路径

同一 image URL 生成稳定 daemon ID；已有 daemon 被直接复用，而不是每个 sandbox 启动一个 FUSE process。[`http.go:259-368`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go#L259) bootstrap 首次仍位于 mount critical path，后续可通过 hardlink cache 复用。[`daemon.go:662-716`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L662) [`bootstrap_cache.go:75-176`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap_cache.go#L75)

这对 same-node same-image rollout 有双重效果：

- 好处：N 个 sandbox 不会产生 N 份 FUSE daemon、bootstrap 和 blob cache；
- 风险：第一次 mount 是共享 barrier，首个 caller 的 registry/bootstrap/FUSE readiness 延迟会一起阻塞等待者。

Nydus mount 暴露的是只读目录，OCI loader 直接把它设置为 `spec.Root.Path`；每个 sandbox 的可写层由 runsc `--overlay2=root:memory` 提供。[`oci_loader.go:136-165`](../../akernel/src/sandboxd/pkg/runtime/oci_loader.go#L136) [`runsc/client.go:75-145`](../../akernel/src/sandboxd/pkg/runtime/runsc/client.go#L75) 所以 AKernel 当前 rootfs 写入不是 RAFS 自己的持久 writable upper，也不是 OverlayBD 的 block upper。

### 3.5 AKernel 的 P2P/Dragonfly：存在代码不等于默认生效

这里必须拆成两条路径。

第一条是 `distill_fs` 自己的 chunk server/peer：源码支持 TCP/Unix socket、static/Redis peer discovery、Redis chunk owner index和 whole-chunk 获取。[`peer.rs:1129-1184`](../../akernel/src/distill-fs/src/backend/peer.rs#L1129) [`peer.rs:1289-1419`](../../akernel/src/distill-fs/src/backend/peer.rs#L1289) [`peer.rs:1866-2053`](../../akernel/src/distill-fs/src/backend/peer.rs#L1866)

但是当前 `sandboxd` 启动 `distill_fs` 的参数只包含 mount/cache/backend/ChunkDB/ImageDB，没有启动 `serve-chunk`，默认 Helm/Terraform 也没有额外 chunk-server 进程。因此当前部署中不能把这条 peer path 当作已接通的跨节点收益。即使后续接通，Nydus uncompressed `ChunkDB` miss 的主路径当前会直接读 backend；peer prefetch 主要位于包裹 blob sparse cache 的 `DedupReader` miss path。[`daemon.go:451-492`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L451) [`dedup.rs:517-550`](../../akernel/src/distill-fs/src/backend/dedup.rs#L517)

第二条是可选 Dragonfly deployment：它默认关闭；启用后让 registry/Nydus/object-storage 流量经过专用 seed-client HTTP proxy 池。[`deploy/README.md:83-95`](../../akernel/deploy/README.md#L83) 这不是“每个 AKernel worker 自带一个 distill peer”，也不能与 `nydus-storage` crates 来自 DragonflyOSS 组织混为一谈。

## 4. AgentENV：OverlayBD-native + ublk + Firecracker 的真实路径

### 4.1 两条 image resolution 分支

~~~mermaid
flowchart TB
    A[userImage] --> B[regctl manifest get]
    B --> C{manifest layer类型}
    C -->|OverlayBD-native| D[生成remote image.json\nrepoBlobUrl,digest,size,dir]
    C -->|标准OCI tar| E{converted commit cache全命中?}
    E -->|是| F[引用本地.commit lowers]
    E -->|否| G[regctl image copy]
    G --> H[逐层untar/apply/转换为.commit]
    H --> F

    D --> I[ImageFile/LSMT]
    F --> I
    I --> J[ublk userspace block device]
    J --> K[Firecracker virtio-blk rootfs]

    K --> L[guest ext4 block read]
    L --> J
    J --> M[LSMT top-down mapping]
    M --> N[ZFile jump table/encoded block]
    N --> O{node remote-block cache/P2P/local commit}
    O -->|miss| P[registryfs_v2 HTTP Range]
~~~

源码在文件头已经清楚写出两条语义：标准 OCI cache miss 会 background `regctl image copy`，但 converter 仍等待各层并按顺序全部转换；native layer 则不下载 blob，运行时按 range 获取。[`oci_image.rs:1-30`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L1)

具体而言：

- Native：`ResolvedImage::Remote` 只保存 registry blob URL 和每层 digest/size/dir。[`oci_image.rs:69-89`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L69) [`oci_image.rs:212-235`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L212)
- Standard OCI：converted layer cache 全命中才跳过下载；否则启动 `regctl image copy`，逐层等待 blob、转换并存入 content-addressed commit cache。[`oci_image.rs:236-275`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L236) [`oci_image.rs:381-438`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L381) [`oci_image.rs:477-508`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L477)

因此对 RL benchmark 来说，测试前必须记录实际走的是 `Remote` 还是 `Local(converted)`，否则同一个“AgentENV create”指标会混入完全不同的镜像准备成本。

### 4.2 从 guest block 到 registry range

OverlayBD lower 是 LSMT layered block image。每个 `DiskSegmentMapping` 用两个 `u64` 打包 logical offset/length、physical offset、zero/tag，共 16 bytes；`ComboIndex` 先查 upper，再用 lower 填补 gap，实现 top-down overlay semantics。[`format.rs:7-55`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/format.rs#L7) [`index.rs:600-667`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/index.rs#L600)

ZFile 使用 jump table 将 logical compressed block index 映射到 encoded offset，按触达块读取、CRC/解压，支持把相邻 encoded blocks 合并成一个 range read。[`zfile.rs:720-839`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L720) [`zfile.rs:962-1040`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L962)

远端 lower 的 URL 由 `repoBlobUrl/digest` 组成；`RegistryFileImplV2` 最终调用 `read_range_into`。[`image_file.rs:631-676`](../../kvcache-ai/AgentENV/storage/overlaybd/src/image/image_file.rs#L631) [`registryfs_v2.rs:1542-1600`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L1542) HTTP backend 明确构造 `Range: bytes=offset-end`，处理 redirect/auth/retry并把 response chunk 流式写入 caller buffer。[`registryfs_v2.rs:704-779`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L704) [`registryfs_v2.rs:825-876`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L825)

ublk target 把这个 `ImageFile` 暴露为 Linux block device；read request 直接进入 `read_at_into_with_ctx`，write 进入 writable upper。[`overlaybd_target.rs:161-223`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L161) 这条路径不经过 host FUSE pathname lookup，guest 自己的 ext4 负责 inode/dentry/file semantics。

“不下载完整layer”不等于“打开镜像时零远端字节”。每个remote lower仍要读取外层tar header，再验证LSMT header/trailer并加载index；lower最多32路并行打开，之后可复用持久化premerged index。[`tar.rs:25-75`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/tar.rs#L25) [`readonly.rs:26-52`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/file/readonly.rs#L26) [`types.rs:11-20`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/file/types.rs#L11) 因而层数多、index大、all-cold节点多时，metadata Range QPS本身也会进入create tail。

### 4.3 节点 remote-block cache 与 background full download

默认生成的 rootfs OverlayBD config 启用 node-local file cache：容量来自 `[image.cache.remote_blocks]`，当前默认 10 GiB；refill unit 为 256 KiB。[`default.toml:54-65`](../../kvcache-ai/AgentENV/config/default.toml#L54) [`deps.rs:598-615`](../../kvcache-ai/AgentENV/src/setup/deps.rs#L598)

remote cache miss 会：

1. 按 fixed block 计算覆盖范围；
2. 用 per-block waiter 合并 concurrent refill；
3. 直接把 source range 写入 sparse cache mmap；
4. 更新 bitmap；
5. 后续读取复用本地块。

证据见 [`cache_store.rs:189-280`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/cache/full_file_cache/cache_store.rs#L189)、[`cache_store.rs:519-636`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/cache/full_file_cache/cache_store.rs#L519) 和 [`cache_entry.rs:87-113`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/cache/full_file_cache/cache_entry.rs#L87)。这使同节点同 blob 的 first-touch 更容易被合并，是 AgentENV 在 same-node rollout 下的重要能力。

另有“完整 layer background download”机制，但 **rootfs 默认关闭**：`[ublk.overlaybd].download_enable = false`。[`default.toml:155-167`](../../kvcache-ai/AgentENV/config/default.toml#L155) 它与默认开启的 **memory snapshot** background download 不是一回事。

rootfs background download 开启后，会等 sandbox envd ready（ready signal 丢失时最多等 20s fallback），然后按块并发下载完整 layer；foreground read 忙时 admission gate 只保留少量 background progress。完成后校验 SHA-256、rename 为 `.commit`，`SwitchFile` 热切到本地并 best-effort 发布 P2P。[`download_gate.rs:1-34`](../../kvcache-ai/AgentENV/storage/overlaybd/src/download_gate.rs#L1) [`bk_download.rs:215-310`](../../kvcache-ai/AgentENV/storage/overlaybd/src/bk_download.rs#L215) [`bk_download.rs:718-840`](../../kvcache-ai/AgentENV/storage/overlaybd/src/bk_download.rs#L718)

它的取舍是：

- foreground 仍可立刻 lazy read；
- 长运行 sandbox 最终转为本地完整 layer，降低后续 remote tail；
- 但最终网络字节趋近完整 layer，并可能与大量 foreground first-touch 争带宽；
- 默认关闭意味着不能把这项收益自动写进 AgentENV rootfs 基线。

### 4.4 writable upper、ublk warm pool 与共享边界

AgentENV 在 immutable compressed lowers 上叠加 writable upper。LSMT read 从 newest mapping 向下补 gap；write 进入本地 upper，而不是修改共享 lower。该模式适合 sandbox filesystem mutation、pause 后 seal/restack，以及把 rootfs/attached drive/memory snapshot 都表示成 OverlayBD layer chain。

ublk daemon 的 warm pool 可以复用已经创建的 ublk device，并通过 `swap_state` 换 backing image，减少 ADD/DEL device setup。[`overlaybd_target.rs:126-158`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L126) [`ublk-daemon/server.rs:124-176`](../../kvcache-ai/AgentENV/storage/ublk-daemon/src/server.rs#L124) 但它只优化 device control path，**不等于镜像数据已经预热**。

源码中的顺序也证明这一点：`AcquireOverlaybd` 先 `create_image_file` 打开目标镜像，之后才从idle pool取device并 `swap_state`；release时还会清device page cache并切到daemon-owned placeholder，以免idle device继续pin业务镜像。[`ublk-daemon/server.rs:1056-1105`](../../kvcache-ai/AgentENV/storage/ublk-daemon/src/server.rs#L1056) [`ublk-daemon/server.rs:1108-1140`](../../kvcache-ai/AgentENV/storage/ublk-daemon/src/server.rs#L1108) [`ublk-daemon/server.rs:1334-1378`](../../kvcache-ai/AgentENV/storage/ublk-daemon/src/server.rs#L1334)

与 AKernel 的“同一 image 共享一个 FUSE mount”不同，AgentENV rootfs runtime device 仍按 sandbox 独占管理；共享主要发生在：

- content-addressed local `.commit`；
- remote-block cache；
- premerged index artifact；
- host page cache（受 direct-I/O/config 影响）；
- P2P 完整 layer artifact；
- memory snapshot 特有的 shared read-only ublk/refcount。

所以不能直接把 DADI 论文“每 container 一个独立 VBD 导致 shared layer page cache 不天然共享”的 2020 限制机械套用到当前 AgentENV，也不能在未实测前声称已经完全消除。DADI 原论文在 PDF 物理第 13 页/印刷 p.738 把 shared block pool/DAX 作为未来方向；AgentENV 的 ublk/cache 架构已经不同，仍应通过 host cache hit 和实际 RSS/IO 指标验证。

### 4.5 AgentENV P2P 与 DADI/FaaSNet 的差别

AgentENV 的全局 `[p2p]` 默认关闭。[`default.toml:67-73`](../../kvcache-ai/AgentENV/config/default.toml#L67) 开启后：

- registryfs foreground range 先请求 localhost P2P HTTP facade；
- facade 根据 `overlaybd-layer/v1/<digest>` 查找完整 layer artifact；
- peer 命中则从 Iroh blob 提取所需 range；
- miss/失败回退 origin registry；
- 完整 background-downloaded layer 才会发布为 peer artifact。

证据见 [`facade.rs:246-299`](../../kvcache-ai/AgentENV/src/overlaybd/p2p/facade.rs#L246)、[`artifact.rs:126-145`](../../kvcache-ai/AgentENV/src/overlaybd/p2p/artifact.rs#L126) 和 [`p2p-design.md:90-128`](../../kvcache-ai/AgentENV/docs/src/internals/p2p-design.md#L90)。

这不是 DADI/FaaSNet 的固定树式 streaming cache：

- partial foreground range 本身不会立即变成一个可发布的 partial layer；
- provider 要先拥有完整 content-addressed layer artifact；
- scheduler 只存 artifact→node hint，不代理 bytes，也不维护 per-image balanced dissemination tree；
- 开启 P2P 时 registry source 走 Direct facade，当前代码不会再套同一层 local file-cache wrapper，因此 peer miss/fallback 下的 node-local partial-cache行为也不同。[`image_service.rs:260-319`](../../kvcache-ai/AgentENV/storage/overlaybd/src/image/image_service.rs#L260)

## 5. Nydus/distill_fs 与 OverlayBD 的机制对比

| 维度 | AKernel Nydus/`distill_fs` | AgentENV OverlayBD |
|---|---|---|
| 数据抽象 | 文件/inode/chunk | logical block/segment |
| metadata | RAFS bootstrap | `image.json` + LSMT header/trailer/index + ZFile jump table |
| host→sandbox接口 | FUSE directory 给 runsc | `/dev/ublkbN` 给 Firecracker virtio-blk |
| guest filesystem | runsc/gVisor看到host FUSE tree | guest内ext4解析block device |
| lazy单位 | RAFS file chunk对应compressed blob range；外层cache为4 MiB | LSMT mapping + ZFile encoded block；node remote cache当前256 KiB refill |
| 压缩 | Nydus blob compressor | ZFile LZ4/Zstd random access |
| read path额外工作 | FUSE lookup/read、RAFS metadata、range、decompress | guest ext4、virtio/ublk、LSMT index、range、decompress |
| writable层 | runsc memory overlay；Nydus mount本身只读 | local LSMT/sparse/hybrid writable upper |
| 节点同镜像共享 | 一个per-image daemon/mount/cache | shared commit/remote-block cache/index；runtime ublk rootfs device按sandbox |
| 跨镜像去重 | global ChunkDB by checksum，能力较显式 | 以OCI/OverlayBD layer digest和block cache复用为主，无等价的全局uncompressed file-chunk DB |
| standard OCI fallback | 没有Nydus variant时完整 pull/extract | 没有native layer/converted cache时完整 pull/convert |
| P2P默认值 | distill peer未接线；Dragonfly默认关闭 | Iroh P2P默认关闭 |
| snapshot统一性 | 当前Nydus path主要解决rootfs | 同一OverlayBD层模型覆盖rootfs、memory、extra drive |
| 格式限制 | 当前只支持RAFS v5；拒绝v6/encrypted/batch | 只接受native/standard两种整齐manifest；拒绝mixed/tar-wrapped等不支持形态 |

### 5.1 类似之处

二者共同采用了以下原则：

1. **metadata-first**：mount/boot前只要求足够的结构 metadata；
2. **fine-grained on-demand transfer**：按 chunk/block range 读取，不以完整 layer 为启动前提；
3. **independently compressed units**：为随机访问牺牲部分压缩率，换取局部解压；
4. **immutable shared lower + private mutable upper**；
5. **content-addressed identity/cache**；
6. **把 cold-start I/O 延后到 first touch**。

最后一点既是优势也是风险：create-ready 变快并不代表 RL environment 的第一步更快。若 Python import、CUDA runtime、browser engine 或仿真器在第一步集中触发大量读取，延迟只是从“spawn阶段”移动到了“first useful action”。

### 5.2 本质差异

文件级 Nydus 的优势是它知道“应用正在读哪个文件的哪个 chunk”，不需要先让 guest filesystem 从远端 block 中解析 inode/extent；同内容 file chunks 也更容易跨镜像 checksum 去重。代价是 pathname/FUSE 边界、RAFS格式限制，以及当前自研 reader 的 worker/串行 chunk loop可能成为热点。

块级 OverlayBD 的优势是对上层 filesystem 和 container/VM 形式更透明，尤其适合 Firecracker：guest 认为自己拥有普通 block rootfs，snapshot和write也能复用 layer machinery。代价是 filesystem metadata read、ext4 read-ahead、block/ZFile边界不对齐可能产生 read amplification；跨镜像“同一文件内容”的语义去重也没有 RAFS ChunkDB 那么直接。

## 6. Google Agent Substrate 是否有类似设计

### 6.1 当前答案：没有 runtime lazy rootfs

Substrate 当前执行：

~~~text
remote.Image
  -> layers
  -> layer.Uncompressed()
  -> download + decompress + untar完整layer
  -> node-local content-addressed unpacked layer pool
  -> overlayfs lowerdirs
  -> actor-private upper/work
  -> gVisor gofer 或 microVM virtiofs lower
~~~

`EnsureImage` 对 digest ref 可无网络命中，对 tag ref 要先 HEAD；cache miss 后每 image 最多 4 个 layer 并行，image-level 和 layer-level singleflight 合并同节点重复工作。[`imagecache.go:256-310`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L256) [`imagecache.go:342-437`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L342)

`unpackLayerToPool` 明确调用 `layer.Uncompressed()` 并完整 untar 到 temp dir，最后 atomic rename。[`imagecache.go:439-503`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L439) 这与 Nydus/OverlayBD “运行后继续按需读”有本质区别。

warm node 上，ateom 只需要 finalise whiteout并做一次 overlayfs mount；每 actor 的 upper/work 私有。[`bundle_linux.go:35-98`](../../agent-substrate/substrate/internal/imagecache/bundle_linux.go#L35) microVM 路径再把这个 composed rootfs bind 到 virtiofsd shared dir，guest 使用 tmpfs upper。[`ateom-microvm/run.go:478-543`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L478)

### 6.2 它现在优化了什么

Substrate 的 node-local layer pool 仍然很有价值：

- layer 以 diffID content-addressed，同节点跨 image/actor 只下载解包一次；
- same-image/layer concurrent pulls有 singleflight；
- warm node compose 无需重新解包；
- shared unpacked lower 可共享 host page cache；
- snapshot download 与 OCI/asset preparation 并行，隐藏两条链路中的较短一条。[`atelet/main.go:556-594`](../../agent-substrate/substrate/cmd/atelet/main.go#L556)

项目 README 报告 `oci_unpack` 从旧路径约 15–20s 降到 warm node 的 single-digit milliseconds。这是**项目文档观察，不是本文复现实测**。[`imagecache/README.md:18-33`](../../agent-substrate/substrate/internal/imagecache/README.md#L18)

### 6.3 当前缺口

- cold node 仍下载和解包完整 layer；
- singleflight 只在单节点内生效；
- scheduler 只按 sandbox class、label、required node 和空闲 assignment 选择 worker，没有 image-cache locality。[`scheduling.go:89-125`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go#L89)
- 文档明确把 lazy materializer、cache-digest reporting、`PreloadImage` 放在后续阶段；
- 当前 GC/eviction 尚未完成，cache disk usage会随历史 layer 单调增长。[`imagecache/README.md:158-178`](../../agent-substrate/substrate/internal/imagecache/README.md#L158)

因此 Substrate 当前更接近“eager pull once per node, then mount many”，而 AKernel/AgentENV native 路径是“mount first, fetch working set on demand”。

## 7. 论文性能证据：能说明什么，不能说明什么

### 7.1 DADI/OverlayBD 论文

论文：Huiba Li 等，*DADI: Block-Level Image Service for Agile and Elastic Application Deployment*，USENIX ATC 2020。[本地 PDF](<../References/Li 等 - 2020 - DADI Block-Level Image Service for Agile and Elastic Application Deployment.pdf>)；[USENIX 官方页面](https://www.usenix.org/conference/atc20/presentation/li-huiba)。

DADI 的系统组合不是单独一个“块格式”，而是：

- block-level layered image；
- fine-grained remote transfer；
- ZFile random-access compression；
- trace-based prefetch；
- tree-structured P2P；
- append-only writable layer。

论文中的关键结果如下：

| 结果 | 论文位置 | 解释与限制 |
|---|---|---|
| 1000 hosts 上 10,000 containers 在 4s 内启动 | 摘要；PDF物理第1页 | 是完整DADI系统，不是当前AgentENV默认配置 |
| WordPress：`.tgz` 165MB、解包501MB、DADI-LZ4 274MB、启动前metadata tar仅9KB | PDF物理第11页/印刷p.736，§5.2 | 说明metadata-first的启动前字节优势 |
| trace prefetch减少 cold/warm 差距的95% | 同页 Figure 16 | AgentENV当前有prefetch框架，但不能据此声称复现同样trace策略；AKernel当前主read path也没有等价的文件工作集prefetch |
| batch 1→32：pseudo-Slacker约1.5→2.3s，DADI约0.7s且基本稳定 | 同页 Figure 17及正文 | 特定WordPress/网络/host配置 |
| production metadata pull：近半hosts不超过0.2s，其余约1s；`.tgz`对多数host超过20s | 同页 Figure 18及正文 | `.tgz`仍已有base dependency layers，完整cold会更慢 |
| 1000 VM×10 containers=10,000，cold通常只比warm慢1–2s | PDF物理第12页/印刷p.737，Figure 20 | 真实large-scale实验 |
| 100,000 containers启动时间近似flat | 同页 Figure 21 | **几十host特殊root-to-leaf拓扑的投影**，不是真实10万container完整实验 |
| warm path NVMe 比overlayfs/LVM好约15–25%，cloud disk超过2× | PDF物理第11页/印刷p.736，Figure 15正文 | 与DADI压缩/I/O path相关，不能直接外推Rust ublk实现 |
| image build比overlayfs快20–40% | PDF物理第13页/印刷p.738，Figure 25 | 不包含commit/compress时间 |

DADI 还指出一个反直觉限制：每 container 的独立 VBD 使同一 shared layer file 可能对应不同 host pages，不能像 overlayfs 那样天然共享 page cache。论文将 shared block pool/DAX 留作未来工作（PDF物理第13页/印刷p.738，§6）。这说明 block interface 的隔离/统一性与 file-page sharing 之间存在取舍。

### 7.2 FaaSNet 论文

论文：Ao Wang 等，*FaaSNet: Scalable and Fast Provisioning of Custom Serverless Container Runtimes at Alibaba Cloud Function Compute*，USENIX ATC 2021。[本地 PDF](<../References/Wang et al_2021_{FaaSNet}.pdf>)；[USENIX 官方页面](https://www.usenix.org/conference/atc21/presentation/wang-ao)。

FaaSNet 对本文最重要的启示是：**lazy read 降低单节点字节量，但不自动消除 N 节点同时访问中心 registry 的瓶颈。** FaaSNet 用三项组合解决 1→N burst：

1. 每 function 一棵动态 balanced binary Function Tree；
2. 512 KiB independently compressed blocks 按需读取；
3. 节点收到 block 后立即向两个 children 流水转发，而不是等完整 layer。

实验环境是 500/1000 VM pool，每 VM 2 CPU、4GB、1Gbps，并维持 free VM pool；**container provisioning latency 不包含 VM cold start**。默认image为758MB PyStan，function自身约运行2s（PDF物理第8页/印刷p.449，§4.1）。

关键数字：

| 结果 | 论文位置 | 口径 |
|---|---|---|
| production trace含712,295次cold start；约57% image pull超过45s，一部分至少80s | PDF物理第4页/印刷p.445，Figure 3正文 | Alibaba Cloud trace |
| burst中central on-demand峰值response约28s、约113s恢复；FaaSNet峰值约6s、28s恢复 | PDF物理第9页/印刷p.450，Figures 11–13 | 28/113是on-demand optimized baseline，不是vanilla docker |
| 8→128 concurrency：FaaSNet相对baseline 13.4×、Kraken 16.3×、on-demand 5×、DADI+P2P 2.8× | PDF物理第10页/印刷p.451，Figure 14 | 平均container provisioning latency |
| FaaSNet第一个5.5s、第128个7.0s，spawn spread 1.5s；on-demand 16.4s，DADI+P2P 19s | PDF物理第11页/印刷p.452，Figure 15 | spread不是绝对启动时间 |
| DADI+P2P全部128完成需22.3s；论文称比FaaSNet的1.5s pipeline span慢14.7× | 同页 | 14.7×是22.3/1.5的特殊口径，不应称“绝对完成时间快14.7×” |
| 1000 VM上2500 functions，全部在5.1–8.3s；on-demand和DADI+P2P timeout | 同页 Figure 17 | 428MB image，每VM 2–3 functions；仍不含VM cold start |
| 512KiB按需块相对regular docker pull减少83.9% network I/O | PDF物理第12页/印刷p.453，Figure 20 | 特定PyStan images；更大block会增加read amplification |

FaaSNet 不能直接替 AKernel/AgentENV 排名，但清楚说明了两件事：

- RL rollout 的 same-image 1→N 模式非常适合做拓扑化 P2P streaming；
- DADI-style lazy block + 一般P2P并非扩展性的终点，root协调、layer-tree创建和中心registry仍可能成为瓶颈。

### 7.3 当前三个仓库没有可比的实测数据

当前源码与文档中没有在相同硬件、相同 image、相同 registry、相同 sandbox ready 定义下比较：

~~~text
AKernel Nydus/distill_fs
vs AgentENV OverlayBD-native/ublk/Firecracker
vs Substrate unpacked-layer/overlayfs
~~~

因此本文不能给出“谁快X%”的结论。可靠结论只能是条件式的，并提出可复现实验。

## 8. RL Rollout 大规模并发 Spawn 分析

### 8.1 必须先区分四种缓存状态

| 缓存状态 | AKernel | AgentENV | Substrate |
|---|---|---|---|
| 同节点 + warm | 复用同一FUSE daemon/mount、blob cache、ChunkDB；后续spawn主要不是image I/O | remote blocks/commits/index已热，ublk/FC pool减少setup；仍有per-VM boot | unpacked layer已在本地，一次overlay mount；可能是三者中rootfs compose最短 |
| 不同节点 + 各自warm | 各节点本地命中 | 各节点本地命中 | 各节点本地命中 |
| 不同节点 + 少数seed | 默认distill peer未接线；可选Dragonfly保护origin但不是per-node树 | P2P开启且已有完整artifact provider时可从peer range；无完整artifact时仍回origin | 当前无P2P/lazy；除非运维提前preload每节点 |
| 所有节点全冷 | 每节点bootstrap+实际working set；Nydus variant缺失则完整OCI | native时每节点metadata+working set；普通OCI则每节点完整copy/convert | 每节点完整image download+decompress+untar |

把“缓存命中”只写成 cold/warm 二值还不够。至少还应记录：

- manifest/bootstrap warm？
- encoded remote block cache warm？
- decompressed file chunk warm？
- full layer commit/unpacked layer warm？
- host page cache warm？
- VM/snapshot memory warm？
- P2P provider是否已有完整artifact？

### 8.2 简化流量模型

设：

- `N`：参与的 cold worker nodes；
- `P`：每节点同时 spawn 的同镜像 sandboxes；
- `I`：完整 compressed image/layer bytes；
- `W_f`：实际触达的 RAFS compressed ranges；
- `W_b`：实际触达的 OverlayBD encoded ranges；
- `A_f/A_b >= 1`：cache/chunk/block边界和read-ahead导致的读放大；
- `M_n/M_o`：Nydus bootstrap 与 OverlayBD metadata/index bytes。

忽略重试、P2P和跨节点cache时，origin egress 近似：

~~~text
AKernel Nydus:
  N × (M_n + A_f × W_f)

AgentENV OverlayBD-native:
  N × (M_o + A_b × W_b)

Substrate eager OCI:
  N × I

AgentENV standard OCI first conversion:
  N × I + N × local conversion I/O
~~~

`P` 不一定线性进入 origin bytes，因为同节点 cache/singleflight 能合并重复读取；但它会进入 FUSE/ublk/CPU queue depth。若所有 sandbox 首次读取高度同步而 cache fill 尚未完成，实际重复请求和 tail 仍取决于合并粒度。

开启“最终完整 background download”后，AgentENV 长期字节可能从 `W_b` 重新趋向 `I`；收益变成提前消除未来 tail，而不是节约总流量。开启有效树式 P2P 后，理想 origin egress 可以更接近少量seed的 `I`，其余流量转为cluster内部，但当前三系统默认部署都不是FaaSNet式per-image balanced tree。

### 8.3 AKernel 在 rollout burst 下的表现推断

优势：

- 同节点同 image 的 rootfs mount 和 FUSE daemon复用；
- first mount只要求bootstrap，不要求完整blob；
- 文件级访问避免下载未触达依赖；
- 节点级ChunkDB可跨 image checksum复用；
- runsc writable upper在内存，短episode写入快。

主要风险：

1. 同image singleflight把重复工作变成首caller barrier；bootstrap/registry慢会使整批等待。
2. 单image daemon 默认有限FUSE workers，而一个read内部又串行遍历多个 RAFS chunks；大量sandbox同步import会排队。
3. miss path同步完成range fetch、buffer allocation和decompress，first-useful-action可能出现长尾。
4. 不同image daemon的cache in-flight不合并；ChunkDB要等第一份完整chunk写入后才帮助其它image，首波仍可能 herd。
5. image多样性高时，per-image process、FUSE workers、backend runtime、LMDB readers/writers和FD数量一起增长。
6. LMDB只允许单写事务；高并发异步chunk store可能竞争writer lock，队列满后drop又降低后续命中。
7. Dragonfly默认关闭，distill peer默认未接线；不能把跨节点扩散视为基线。
8. 当前按 image URL/tag缓存检测、daemon和bootstrap；mutable tag存在旧mount/旧bootstrap风险，生产和实验应pin digest。[`nydus_cache.go:24-44`](../../akernel/src/sandboxd/pkg/imagemanager/api/nydus_cache.go#L24)

### 8.4 AgentENV 在 rollout burst 下的表现推断

优势：

- native image resolution只生成remote descriptors；
- block interface直接匹配Firecracker rootfs；
- node 级256KiB remote-block cache能够合并相同block refill；
- LSMT/ZFile支持局部random access和encoded block coalescing；
- ublk、network、Firecracker pools减少非镜像setup；
- P2P已有完整layer provider时，foreground可从peer取range；
- 若从golden snapshot resume，memory ublk共享是额外加速项。

主要风险：

1. 普通 OCI 首次使用完全不具备runtime lazy效果，完整pull/convert可能压倒其它差异。
2. all-cold nodes即使只按需读，也会同时向registry发大量Range/TLS/auth请求。
3. guest ext4 metadata/read-ahead、LSMT/ZFile边界、remote-cache 256KiB refill可能造成小文件读放大。
4. ublk daemon是单进程集中承载devices和background downloads；高并发会考验queue/io_uring/CPU而不仅是网络。
5. rootfs background full download默认关闭；开启后会与大量first-touch竞争带宽，虽然gate优先foreground。
6. P2P默认关闭，且只在完整layer artifact已发布时避免origin；partial first touches不会像FaaSNet那样边收边成为下游seed。
7. 每个microVM仍需vCPU/RAM和Firecracker boot/resume资源；image lazy不能消除KVM/内存容量瓶颈。

### 8.5 Substrate 在 rollout burst 下的表现推断

优势：

- warm node rootfs没有remote first-touch，执行期间tail较稳定；
- same-node layers只下载/解包一次；
- image/layer singleflight避免同节点重复pull；
- unpacked overlayfs lower天然适合host page-cache sharing；
- worker pods预热且Actor resume绕过per-Actor Kubernetes Pod创建。

风险：

- all-cold node burst总流量仍近似 `N×I`；
- 每节点还要消耗decompress+untar CPU和大量小文件写入/metadata IOPS；
- scheduler不感知image cache，可能把Actor放到cold node；
- 无跨节点P2P或lazy backend；
- cache当前无完整GC，长期运行有磁盘容量风险。

对于“episode 很短、image 很大、working set 很小”的 rollout，Substrate cold-node最不利；对于“节点长期warm、episode执行中会遍历大部分image”的 rollout，它把远端I/O提前完成，反而可能得到最稳定的step-time尾延迟。

## 9. AKernel + AFaaS Seed Tree 与 Substrate Golden Template

### 9.1 先给结论：二者都复用初始化，但选择了不同的资源—延迟点

OSDI 2025论文 *Fork in the Road* 中AFaaS的seed tree和Agent Substrate的Golden Actor/Golden Snapshot表面上都在做“先运行一次，再从预热状态创建实例”，但它们不是同一种机制：

- **AFaaS seed tree**是每个计算节点内的live、paused、可fork执行状态。实例从最深的可用seed直接派生，父子匿名内存页通过CoW共享；它保留一部分seed内存，换取极短的节点内实例化路径。
- **Substrate Golden Snapshot**是每个`ActorTemplate`的一份持久checkpoint。Golden Actor完成初始化后被Suspend并释放Worker；普通Actor在任意合适Worker上下载并恢复这份checkpoint。它接受更长的download/restore路径，换取资源可完全释放和跨节点可移植性。
- **AKernel当前只适合把seed tree写成后续设计。** 开源v0.1.0明确把“40 ms fork cold start”和Checkpoint/Restore标为planned；`StartRequest.template_id`只为下游兼容而保留，开源runtime会忽略；`RealRuntimeHandler`也没有Fork/Checkpoint/Restore方法。[`README.md:35-39`](../../akernel/README.md#L35) [`sandbox-api.proto:87-94`](../../akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto#L87) [`handler.go:32-43`](../../akernel/src/sandboxd/pkg/runtime/handler.go#L32)

因此本节的比较对象应准确写成：

~~~text
建议中的 AKernel + AFaaS-style node-local seed tree
vs
当前源码中的 Substrate ActorTemplate + Golden Actor + Golden Snapshot
~~~

它也与前文image lazy load正交：**Nydus解决“rootfs bytes何时进入节点”，seed/golden解决“OS、语言runtime与user code初始化是否要重做”。** 只做前者，Python import、JVM/JIT、framework初始化仍可能成为TTUA主项；只做后者，冷节点仍可能在seed构建或snapshot restore时承受image I/O。

### 9.2 AFaaS seed tree到底如何工作

AFaaS不是从一个全功能模板平铺复制所有函数，而是在每个计算节点维护三级树。论文Figure 10的结构可概括为：

~~~mermaid
flowchart TD
    S0[Level 0: root seed\nguest OS初始化状态\n每个compute node唯一]
    S0 --> PY[Level 1: Python seed]
    S0 --> JS[Level 1: Node.js seed]
    S0 --> JVM[Level 1: JVM seed]
    PY --> EA[Level 2: env-A/user-code seed\nimports + framework init]
    PY --> EB[Level 2: env-B/user-code seed]
    EA --> A1[rollout instance A1]
    EA --> A2[rollout instance A2]
    EB --> B1[rollout instance B1]
~~~

构建和命中路径如下：

1. Level 0 root seed完成guest OS初始化后暂停；同一物理节点只保留一个root seed。
2. 每个language/runtime seed从root fork，只执行Python、Node.js或JVM等增量初始化，然后暂停成为Level 1 seed。
3. user-code seed从对应language seed fork，完成framework加载、library import、JIT/compile与函数初始化，然后暂停成为Level 2 seed。
4. 请求到来时先查最深的精确user-code seed；未命中则退回language seed，再无则退回root seed。实例从“nearest viable ancestor”fork，并补做尚未完成的初始化。
5. fork时复制父seed的EPT映射并将末级页表标为只读（EPT Prefill），让父子共享guest physical memory，在写入时才产生私有页。

这里的fork对象不是传统host Linux application process，也不是Firecracker microVM。AFaaS基于Catalyzer/gVisor式secure container，fork的是其sandbox/VM-process与guest OS状态；AKernel虽然同样使用gVisor/runsc，但不能直接调用stock `runsc fork`，需要扩展底层runtime和host控制路径。

AFaaS的低延迟也**不只来自seed tree**。完整系统还使用：

- FRI专用`create/fork/activate`路径，绕过每请求OCI runtime binary加载和多层RPC；
- 预创建的veth与cgroup pool；
- host network/IPC namespace共享和可共享/实例私有network stack拆分；
- 预编译seccomp规则、EPT Prefill和early destroy。

所以论文数字只能作为“完整AFaaS设计的参考”，不能写成“给AKernel加一棵seed tree就必然得到同一数字”。

### 9.3 AFaaS论文给出的性能与内存证据

论文在一台24核Intel Xeon Platinum 8163、512 GiB内存、Linux 5.10服务器上测试。高并发实验中：

| 指标 | AFaaS | CataOnly | 证据边界 |
|---|---:|---:|---|
| E2E latency区间 | 16.34–39.56 ms | 51.32–117.92 ms | JS workload，并发从1增至24；论文Figure 12，PDF物理p.11–12/印刷p.508–509 |
| cold-start区间 | 6.97–14.55 ms | 38.39–74.05 ms | 同上；不是AKernel或Substrate结果 |
| 8个生产Node.js函数整体加速 | 1.80×–8.14× | baseline | AFaaS取生产日均E2E，CataOnly为同硬件mock peer复测，并非严格在线A/B |
| 生产函数startup | 5.45–9.41 ms | 未给统一区间 | Figure 18，PDF物理p.13/印刷p.510 |

内存方面，同user-code seed的树结构相对CataOnly降低28.11%–84.91%的seed memory；生产数据中Level 0为14.31 MiB，Node.js/Python Level 1分别为54.01/29.05 MiB，Level 2约6.90–135.03 MiB（Figure 17与Table 3，PDF物理p.13/印刷p.510）。这些数字衡量seed结构，不是整机RSS/PSS、page cache、EPT metadata、swap与资源池总开销。

还要注意三项扩展性限制：

1. 一个seed同一时刻只能串行clone一个实例，超高并发需要同类型seed副本；论文没有实现dynamic seed scaling。
2. user-specific seeds过多会造成内存压力乃至swap，进而破坏低尾延迟。
3. 论文实验主要是单机、最高约24并发，没有证明跨节点remote fork、跨节点CoW或万级集群扩展。

### 9.4 AKernel当前源码与“seed”这个名字的边界

AKernel当前执行路径仍是每个sandbox独立生成OCI bundle，然后依次调用`runsc create`和`runsc start`。[`runsc_handler.go:144-221`](../../akernel/src/sandboxd/pkg/runtime/runsc_handler.go#L144)

`runsc create`使用的`--overlay2=root:memory`会给每个sandbox建立私有的内存文件系统upper；它可以共享不可变rootfs lower和host cache，但这不是父子sandbox之间的anonymous-memory CoW，也没有形成可递归fork的执行状态树。[`client.go:107-145`](../../akernel/src/sandboxd/pkg/runtime/runsc/client.go#L107) [`oci_loader.go:183-225`](../../akernel/src/sandboxd/pkg/runtime/oci_loader.go#L183)

仓库中虽然存在`LangRTManager.SeedInfo`，它现在只保存`Cid/Cmd/Envs`元数据，不包含可fork的live process、guest memory或页表句柄；`rootfsMap`做的是共享rootfs创建的singleflight与引用复用，不是执行内存CoW。[`manager.go:25-44`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L25) [`manager.go:49-87`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L49) [`manager.go:210-226`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L210)

AKernel论文草稿定义了`Running/Checkpointed --fork--> Created(child)`和AProc/APlane lineage，但设计章节仍保留“确认Fork究竟使用CoW、checkpoint clone还是重新物化”的TODO。因此它是目标抽象，不是当前源码实现。[`4-abstractions.tex:31-41`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/4-abstractions.tex#L31) [`5-design.tex:85-90`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex#L85)

这三个容易混淆的概念应严格区分：

| 名称 | 当前AKernel是否存在 | 是否复用执行内存 |
|---|---|---|
| Nydus/distill_fs rootfs共享 | 是 | 否；共享只读文件数据、FUSE mount和chunk cache |
| `LangRTManager.SeedInfo` | 是 | 否；当前是启动元数据 |
| AFaaS-style paused live seed | 否，建议实现 | 是；目标是父子guest memory CoW |

### 9.5 建议的AKernel接入方式：live fork快路径 + durable snapshot回退

不建议把seed tree做成一棵跨集群共享live memory的“全局树”。AFaaS的root seed是每物理节点唯一，论文没有跨物理节点remote fork。更适合AKernel的设计是：控制面只维护目录，live seed和fork动作都留在节点内。

~~~mermaid
flowchart TB
    CP[AKernel control plane\nseed catalog: key/node/version/health/locality]
    SCH[Scheduler\ncapacity + nearest-seed affinity]
    CP --> SCH
    SCH --> SM

    subgraph NODE[AKernel worker node]
      SM[SeedManager]
      R[root seed\nkernel + gVisor ABI]
      L1[Python runtime seed]
      L2[RL env/template seed]
      I1[AProc rollout 1]
      I2[AProc rollout 2]
      R -->|ForkSeed + init + pause| L1
      L1 -->|ForkSeed + import/init + pause| L2
      L2 -->|ForkSeed + rebind + activate| I1
      L2 -->|ForkSeed + rebind + activate| I2
    end

    AP[APlane durable checkpoint\nrebuild/migration/fallback]
    AP -. seed失效或跨节点miss .-> SM
~~~

建议的数据与控制接口至少包括：

- runtime接口：`PrepareSeed`、`ForkSeed`、`PauseSeed`、`DestroySeed`、`ListSeeds`；不能只在上层把`StartSandbox`换个名字。
- seed key：kernel、gVisor/runsc版本、rootfs digest、language/runtime digest、user-code/template digest、CPU feature、security/network policy和设备兼容性。
- node-local `SeedManager`：引用计数、并发clone门、健康检查、版本失效、LRU/benefit-aware GC、失败重建和副本扩缩。
- cluster seed catalog：只记录哪个节点有哪些兼容seed及其负载，不传输live memory bytes；scheduler在capacity/fairness约束下优先最近的可用ancestor。
- fast path：直接节点内`ForkSeed → per-instance resource/identity rebind → activate`；跨节点或seed失效时从APlane durable checkpoint恢复，或从较浅seed补做初始化。
- 预池：veth/cgroup/namespace等fork后必须私有或重绑定的资源，避免大量fork重新进入慢的通用创建路径。

一次spawn的建议流程是：

~~~text
Resolve immutable template digest
  -> scheduler选择“有最深兼容seed且容量足够”的node
  -> SeedManager锁定某个seed replica
  -> ForkSeed（CoW）
  -> 重建AProc identity / cgroup / veth / credentials / random state
  -> 挂接APlane私有write layer
  -> Activate并等待readyz
  -> 返回first-useful-action可用，而不只是fork完成
~~~

对于exact Level 2 miss，应允许回退Level 1或Level 0，而不是阻塞等待新seed构建。后台可根据`arrival_rate × saved_init_time - resident_memory_cost`决定是否把本次初始化结果提升为新seed。由于AFaaS已暴露“单seed串行clone”限制，AKernel还需按burst并发复制同类型seed，并让scheduler观察seed replica queue，而不是只看节点空闲CPU/RAM。

### 9.6 Substrate Golden Actor/Golden Snapshot的实际源码路径

Substrate的“gold template”更准确地说是`ActorTemplate + Golden Actor + Golden Snapshot`三部分：

~~~mermaid
flowchart LR
    AT[ActorTemplate CRD]
    C[ActorTemplateReconciler]
    GA[普通Golden Actor\nate-golden Atespace]
    W[普通warm Worker Pod\nateom运行sandbox]
    GS[(Object store\nGolden Snapshot)]
    A1[Actor A独立restore]
    A2[Actor B独立restore]

    AT --> C
    C -->|CreateActor + ResumeActor| GA
    GA -->|分配一个free Worker| W
    W -->|cold boot + readyz或等待20s| C
    C -->|SuspendActor / external checkpoint| GS
    GS -->|download + decompress + restore| A1
    GS -->|download + decompress + restore| A2
~~~

源码中的完整流程是：

1. `ActorTemplateReconciler`确保系统`ate-golden` Atespace存在，为模板创建一个普通logical Actor；Golden Actor名称取ActorTemplate UID。[`actortemplate_controller.go:80-116`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L80)
2. controller调用普通`ResumeActor`。因为Golden Actor没有自己的snapshot、模板也还没有golden snapshot，ateapi在普通free Worker上从ActorTemplate spec cold boot。[`actortemplate_controller.go:118-143`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L118) [`workflow_resume.go:70-106`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L70)
3. 每个container都有`readyz`时，Resume已等待到HTTP 200，额外warm-up为0；否则controller固定等待20秒。这里不存在独立`WarmActor` RPC或自动识别模型加载完成的机制。[`actortemplate_controller.go:35-45`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L35) [`actortemplate_controller.go:195-210`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L195)
4. controller调用普通`SuspendActor`；atelet/ateom checkpoint、zstd压缩并上传对象存储，controller只把返回的snapshot name写入`ActorTemplate.status.goldenSnapshot`，然后标记Ready。[`actortemplate_controller.go:145-182`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L145) [`workflow_suspend.go:114-176`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L114)
5. 普通Actor首次Resume的选择优先级是`Actor own LatestSnapshot > Template GoldenSnapshot（boot=false）> cold boot`；从golden启动并产生自己的latest snapshot后，后续恢复沿Actor自己的状态继续。[`workflow_resume.go:81-106`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L81)

`SnapshotsConfig.OnCommit`的scope决定Golden Snapshot的语义。[`actortemplate_types.go:253-293`](../../agent-substrate/substrate/pkg/api/v1alpha1/actortemplate_types.go#L253)

- **Full**：包含process memory、OCI rootfs delta与DurableDir，才能跳过语言/user-code内存初始化；gVisor走`runsc checkpoint/restore`，microVM走Cloud Hypervisor snapshot/restore。
- **Data**：只保存DurableDir；Resume仍要cold boot workload，所以不能称为“预热内存模板”。

Golden Actor不是特殊Pod、专用runtime或常驻父进程。Suspend完成后Worker assignment被释放，后续每个Actor分别把external snapshot下载到自己的restore目录，再启动独立runtime。当前源码没有`ForkActor` RPC、parent/children tree、nearest-ancestor search或跨Actor匿名内存CoW。Substrate虽支持从tagged snapshot创建Actor，但本质仍是one-shot checkpoint clone，不是live fork。

### 9.7 逐项对比

| 维度 | 建议中的AKernel + AFaaS seed tree | 当前Substrate Golden Template |
|---|---|---|
| 复用对象 | paused live sandbox/VM-process及guest memory | object store中的Actor checkpoint |
| 拓扑 | root → language → user/environment多层树 | 每ActorTemplate一份Golden Snapshot，逻辑flat star |
| 基线生成 | 从父seed fork，执行增量初始化，再pause | Golden Actor cold boot，readyz/20s settle，再Suspend |
| 实例化 | 从最深可用ancestor节点内direct fork | 为每Actor下载、解压并restore同一snapshot |
| 精确模板miss | 可回退language/root seed并补初始化 | 无golden时从ActorTemplate cold boot；没有跨模板language ancestor |
| 活跃实例内存 | 同一节点同lineage通过CoW共享未修改页 | 每Actor独立runtime/anonymous memory；没有源码保证的跨Actor内存CoW |
| idle基线资源 | seed持续占节点内存，通常不占运行CPU | Golden Actor释放Worker；只保留snapshot storage/metadata |
| 跨节点 | live seed节点本地；每节点构建或重建 | snapshot可在任意兼容Worker恢复 |
| 持久性 | 节点故障会丢live seed，需要重建/持久fallback | object-store checkpoint天然耐Worker释放和节点故障 |
| burst瓶颈 | seed clone串行门、seed副本数、host cgroup/veth/CoW page fault | free Worker Pod数、object-store fan-out、download/decompress、restore/page fault |
| 后续有状态生命周期 | 更适合从共享基线反复产生短命分支；需另接APlane checkpoint | 首次用golden；此后优先用Actor自己的latest snapshot |
| image路径 | Nydus可为seed构建/实例first-touch按需供数 | 当前先完整拉取/解包OCI layer；snapshot restore还要准备runtime assets |
| 控制路径 | 需要节点内FRI式专用fork/activate fast path | ateapi lock/scheduler → atelet → object store → ateom restore |
| 当前成熟度 | AKernel尚未实现，属于设计建议 | Golden lifecycle已有源码；仍有cleanup/failure TODO |

一句话概括：**seed tree像“保留一组分层、可立即分叉的内存祖先”，Golden Snapshot像“保存一个可搬运、可重复恢复的初始化完成镜像”。** 前者优先本地时延和共享密度，后者优先资源回收、跨节点和stateful Actor连续性。

### 9.8 RL rollout下的条件式性能判断

设：

- `N`为同时启动的rollout数；
- `F`为本地fork与最低限度activate成本；
- `R`为单Actor的snapshot download、decompress、runtime restore与post-restore fault成本；
- `I_miss(k)`为从第`k`层ancestor补做的初始化；
- `Q_seed`为seed replica串行clone排队；
- `Q_worker`为Substrate free Worker不足的排队/失败成本。

忽略调度与网络抖动时，命中路径的单实例关键时间可粗略写成：

~~~text
AKernel seed hit:       F + Q_seed + post-fork rebind + readyz
AKernel ancestor miss:  F + Q_seed + I_miss(k) + readyz
Substrate golden Full:  R + Q_worker + readyz
Substrate golden Data:  R_data + Q_worker + application cold initialization + readyz
~~~

这给出几个可检验判断：

1. **同节点、同环境、短episode、高burst**：精确Level 2 seed命中时，AKernel路径更可能最低延迟且内存密度更高；公共Python/framework页通过CoW共享。代价是要预留seed和足够副本，且所有实例落在有seed的节点。
2. **节点频繁伸缩或故障、跨节点调度、长idle**：Substrate更稳妥。Golden Snapshot不依赖源Worker存活，可把所有运行CPU/RAM释放后在别处恢复；AKernel若没有durable fallback就必须重建整棵局部树。
3. **大规模all-at-once fan-out**：AKernel风险是单seed串行clone queue与clone后资源锁争用；Substrate风险是`N`个free Worker和对同一checkpoint的`N`次独立download/decompress/restore。当前Substrate没有FaaSNet式tree propagation或共享snapshot-memory fast path。
4. **环境种类非常多、每类只执行一次**：Level 2 seeds的准备和驻留不一定回本；Substrate按模板存snapshot也会放大对象存储，但不会持续占Worker内存。两者都应按热度/收益建模板，而不是为每个唯一任务无条件预热。
5. **长episode或执行占主导**：两种startup优化对总makespan影响都会下降；应优先比较steady-state step time、CPU/RSS和状态正确性。

对RL系统最重要的不是`fork()`返回时间或`ResumeActor` RPC返回时间，而仍是本文定义的TTUA：

~~~text
TTUA = request arrival
     -> image/rootfs可读
     -> sandbox执行状态可用
     -> environment/policy初始化完成
     -> 第一条有效transition完成
~~~

seed预热时还应主动触达预计工作集。否则父seed只“逻辑完成初始化”但Nydus chunk未热，许多child会在相同first action同步触发远端读取，fork虽然快，TTUA p99仍会被first-touch拖长。

### 9.9 正确性、隔离与运维代价

live fork与Full snapshot都会复制不应跨实例复用的状态。至少需要在fork/restore后重新生成或重新绑定：

- actor/AProc ID、hostname、MAC/IP、cgroup和审计identity；
- `getrandom`缓冲、PRNG state、TLS/session token和临时credentials；
- 数据库/socket连接、租约、timer/clock基准与外部服务session；
- per-rollout writable APlane layer、secret引用和network policy。

AFaaS论文明确报告gVisor buffered `getrandom`可能让fork实例产生相同随机序列；这是正确性和安全问题，不是可选优化。Substrate也有类似风险：Full Golden Snapshot可能冻结Golden Actor启动时解析的secret和连接，Secret更新不会自动重建snapshot。实例特有状态应通过恢复后的fresh mount/metadata channel注入，而不是烘焙进共享内存。

运维上还需处理：

- seed版本更新与旧代码/旧runtime失效；
- 节点故障后的树重建和防stampede；
- seed memory quota、swap避免、按收益GC；
- 单seed串行fork时的副本扩缩；
- Golden Snapshot对象存储GC与ActorTemplate删除清理；
- snapshot/fork失败后的Worker或seed占用泄漏；
- CPU feature、kernel/runtime ABI与设备兼容验证。

Substrate当前controller源码已经留有两个相关TODO：Golden Actor Resume失败可能泄漏被占用Worker，Suspend conflict也缺少完整补偿；ActorTemplate删除分支当前直接返回，不由该controller清理Golden Actor/snapshot。[`actortemplate_controller.go:75-78`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L75) [`actortemplate_controller.go:120-123`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L120) [`actortemplate_controller.go:152-154`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L152)

### 9.10 建议增加的head-to-head实验

在本文第11节通用image benchmark之外，再固定同一Python RL environment，增加以下四条启动路径：

1. AKernel现有`runsc create/start + Nydus`；
2. AKernel Level 0/Level 1/Level 2 seed分别命中；
3. Substrate cold boot、Data Golden和Full Golden；
4. Substrate Full Golden在same-node、cross-node、snapshot local-cache cold/warm下分别测试。

除`p50/p95/p99 TTUA`外，必须报告：

- seed fork/activate、ancestor补初始化、post-fork rebind各阶段时间；
- snapshot metadata、download、zstd decompress、runtime restore、post-restore page fault时间；
- 1/32/128/1000并发下seed queue与free Worker queue；
- 每个运行rollout的RSS/PSS、CoW shared/private pages、host page cache、seed resident memory和snapshot storage；
- node failure后重建/恢复时间，以及模板版本更新后的失效时间；
- first useful step触发的Nydus远端bytes、Range数和同步first-touch tail；
- identity、entropy、secret、network connection与writable-state隔离测试。

AFaaS的6.97–14.55 ms只能作为研究目标线；公平实验必须让AKernel seed方案和Substrate Golden方案使用同样的CPU/内存限制、镜像内容、readyz语义、节点缓存状态和TTUA终点。

## 10. 哪一种更适合典型 RL Rollout

### 10.1 同一 environment/policy image，短episode，工作集很小

理想条件下 AKernel/AgentENV native 都明显优于 cold-node eager pull。AKernel更可能减少无关文件数据；AgentENV更容易从snapshot恢复完整microVM状态。决定胜负的指标不是create ACK，而是：

~~~text
TTUA = request arrival -> environment完成第一步有效transition
~~~

如果第一步会密集import，AKernel单daemon FUSE queue和AgentENV registry range storm都可能造成p99。

### 10.2 同一镜像，多episode长期驻留在少量节点

三者都会逐渐变warm。此时：

- AKernel共享一个FUSE mount，rootfs复用直接；
- AgentENV block/device/Firecracker pool和snapshot路径可能整体更强；
- Substrate已解包overlay layers，运行时I/O路径最传统，可能有最稳定的warm performance。

需要测 CPU/IOPS/RSS，而不是继续用cold-start数字判断。

### 10.3 每个rollout镜像不同或依赖高度多样

这是所有content cache的困难场景。AKernel会产生更多per-image daemon/threads并竞争共享LMDB；AgentENV会生成更多index/remote cache/ublk states；Substrate会快速增长unpacked layer pool。此时跨image layer/chunk重复率、GC和scheduler locality比单次lazy机制更重要。

### 10.4 大模型权重与rootfs

不要假设把数十GB policy weights塞进rootfs就能自动获得理想lazy性能。权重通常顺序读取、使用率高，并可能被GPU direct I/O、mmap和page locking放大。更合理的分层是：

- rootfs lazy image：OS、Python/runtime、environment binary；
- 独立只读model artifact/cache：按模型digest预置或拓扑化分发；
- per-episode mutable state：snapshot/COW upper；
- replay/result：独立持久数据通道。

## 11. 建议的可复现实验

### 11.1 公平比较的部署矩阵

至少准备以下三条可控路径：

1. AKernel：确认每个 image digest确实有 RAFS v5 Nydus variant；
2. AgentENV：发布同内容的 OverlayBD-native image，另测 standard OCI conversion作为独立case；
3. Substrate：相同逻辑内容的标准 OCI image。

镜像内容应一致，但格式不同导致compressed size不同，必须同时报告：

- source compressed bytes；
- metadata/bootstrap/index bytes；
- unpacked/virtual size；
-实际工作集和远端读取bytes。

### 11.2 因子设计

| 因子 | 建议取值 |
|---|---|
| 节点缓存 | all-cold / metadata-warm / partial-warm / full-warm |
| 并发 | 1 / 32 / 128 / 1000 / 10000（逐级，先验证容量） |
| placement | 1 node×N、N nodes×1、N nodes×P |
| working set | 1% / 10% / 50% / 100% |
| access pattern | import/small random、large sequential、metadata-heavy、write-heavy |
| image diversity | 1 image / 10 images / per-sandbox unique |
| P2P | off/on；分别记录provider预存在/不存在 |
| snapshot | fresh boot / golden snapshot resume，作为正交因子 |

对 AKernel 额外测：ChunkDB on/off、Dragonfly off/on、single image vs many images。对 AgentENV 额外测：native vs standard OCI、remote cache cold/warm、rootfs background download off/on、Iroh off/on。对 Substrate 额外测：cache-aware manual pinning vs random placement，以及preload完成/未完成。

### 11.3 时间指标

不要只记录一个“startup latency”，应分段：

~~~text
T0  request accepted
T1  image metadata/bootstrap resolved
T2  rootfs mount/block device ready
T3  runtime/VM started
T4  envd/readyz ready
T5  first target file open completed
T6  model/environment initialized
T7  first useful RL transition completed
~~~

核心报告：`p50/p95/p99/max(T4-T0)`、`p50/p95/p99/max(T7-T0)` 以及 `T7-T4`。如果只报 `T4`，lazy系统可能看起来很快，却把所有损失隐藏到首次action。

### 11.4 资源与放大指标

- origin registry requests/s、Range count、bytes、TLS/auth latency；
- P2P internal bytes、origin fallback bytes、provider fan-out；
- host network RX/TX、disk read/write/IOPS；
- FUSE queue/wait/worker CPU、RAFS/cache/ChunkDB hit；
- ublk queue depth/io_uring latency、LSMT/ZFile decompress CPU、remote-block cache hit；
- Substrate download/decompress/untar时间与bytes；
- read amplification：remote encoded bytes / application requested bytes；
- per-node RSS/page-cache/cgroup memory；
- first-wave与steady-state差异；
- failure/retry/timeout与tail correlation。

### 11.5 必须控制的陷阱

1. 所有image使用digest pin，避免mutable tag污染。
2. 冷测同时清理项目cache和kernel page cache；温测明确哪一层保留。
3. registry、proxy和workers网络限速一致。
4. VM/container资源一致；FaaSNet风格实验要明确是否排除worker/VM cold start。
5. 不把AgentENV standard OCI conversion混进native lazy基线。
6. 不把AKernel普通OCI fallback混进Nydus基线。
7. P2P test记录seed是否已有完整artifact；不能只写开关on/off。
8. 大并发先做capacity admission，避免把CPU/RAM不足误判为image system问题。

## 12. 面向三个项目的改进建议

### 12.1 AKernel

优先级从高到低：

1. 镜像必须digest pin，并让daemon/bootstrap identity包含manifest digest；
2. 把“是否存在Nydus variant”变成发布/调度前校验，避免运行时silent eager fallback；
3. 公开FUSE queue、backend range、decompress、RAFS/4MiB cache、ChunkDB/LMDB metrics；
4. 在same-image burst下增加/自适应FUSE worker，并评估一个request跨chunk并行/预取；
5. 真正接通并验证distill peer，特别验证uncompressed RAFS chunk miss是否能够跨节点命中；
6. 将image/cache locality纳入scheduler；
7. 对固定rollout模板增加显式working-set recording/prefetch；
8. Dragonfly开启时压测seed-client池和Range缓存语义，不把“P2P已部署”当作充分证明。

### 12.2 AgentENV

1. 为常用RL images建立OverlayBD-native发布pipeline，避免首次standard OCI conversion；
2. scheduler heartbeat报告image/layer/remote-block cache locality；
3. 区分partial-block cache provider与full-layer P2P artifact，探索边收边服务或FaaSNet式拓扑传播；
4. 在大规模burst下验证P2P Direct模式跳过local remote-block cache的取舍；
5. 对rootfs background download做基于episode lifetime/working set的自适应启用，而非固定开关；
6. 报告ublk queue、remote cache、ZFile read/decompress、origin/P2P source和first-action metrics；
7. 将rootfs、memory snapshot和model artifact分别计量，避免把memory resume收益归因给image lazy。

### 12.3 Agent Substrate

低风险演进顺序：

1. 全面使用digest pin；
2. Worker上报cached image/layer digests；
3. scheduler加入cache affinity但保留capacity/fairness约束；
4. 实现`PreloadImage`与可观察的完成状态；
5. 完成watermark GC；
6. 通过现有materializer seam接入eStargz/SOCI/Nydus一类lazy backend；
7. all-cold多节点仍不够时，再加入P2P block streaming，而不是只做full-layer peer copy。

## 13. 最终判断

AKernel Nydus/`distill_fs` 和 AgentENV OverlayBD 都实现了“用工作集替代完整镜像作为启动关键路径”的核心思想，但它们服务于不同runtime边界：

- AKernel把优化放在**文件语义**上，并通过共享FUSE mount和ChunkDB加强同节点复用；
- AgentENV把优化放在**块设备语义**上，并将它与Firecracker、writable upper、snapshot/drive统一；
- Substrate当前选择**完整解包一次、节点内反复mount**，cold-node较重但warm执行路径稳定。

对于 RL rollout，大规模并发spawn最可能暴露的不是单机格式解析速度，而是：

~~~text
all-cold node fan-out
  + synchronized first-touch
  + origin/P2P topology
  + decompression/metadata CPU
  + runtime/VM capacity
  + first useful action tail
~~~

DADI证明了块级lazy的潜力；FaaSNet则提醒我们，真正把1→N burst扩展到大规模，需要把按需读取和拓扑化传播联合设计。当前 AKernel 与 AgentENV 都已经具备做这类演进的基础，但两者默认部署中的跨节点传播能力都还不能等同于FaaSNet。最终选择应由本文第11节的同机实验决定，而不是由格式名称决定。

进一步加入执行状态复用后，建议中的AKernel seed tree与Substrate Golden Snapshot不是互相替代的单选项：前者适合作为命中同节点热模板时的live fork fast path，后者代表可释放资源、可跨节点恢复的durable fallback。AKernel更合理的长期形态是用Nydus按需提供不可变rootfs、用node-local seed tree压缩热门rollout的初始化路径，再用APlane checkpoint覆盖节点故障、长idle和跨节点迁移；不能只实现其中一层便宣称解决了完整cold start。

## 14. 关键源码索引

### AKernel

- 当前sandboxd API无Fork/Checkpoint/Restore，`template_id`被忽略：[`sandbox-api.proto:21-35`](../../akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto#L21) [`sandbox-api.proto:87-95`](../../akernel/src/sandboxd/api/runtime/v1/sandbox-api.proto#L87)
- 当前runtime逐实例`runsc create/start`：[`handler.go:32-43`](../../akernel/src/sandboxd/pkg/runtime/handler.go#L32) [`runsc_handler.go:200-221`](../../akernel/src/sandboxd/pkg/runtime/runsc_handler.go#L200)
- 当前`SeedInfo`仅为元数据，rootfs共享不是内存seed：[`manager.go:25-29`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L25) [`manager.go:47-88`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L47)
- Nydus路由与fallback：[`http.go:428-545`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go#L428)
- 标准Nydus annotation/bootstrap：[`bootstrap.go:31-166`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go#L31)
- per-image daemon与bootstrap关键路径：[`daemon.go:451-492`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L451) [`daemon.go:662-716`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L662)
- Nydus crates固定版本：[`distill-fs/Cargo.toml:29-32`](../../akernel/src/distill-fs/Cargo.toml#L29)
- RAFS load与v5限制：[`image/nydus.rs:169-231`](../../akernel/src/distill-fs/src/image/nydus.rs#L169)
- RAFS chunk read：[`image/nydus.rs:516-573`](../../akernel/src/distill-fs/src/image/nydus.rs#L516) [`image/nydus.rs:647-684`](../../akernel/src/distill-fs/src/image/nydus.rs#L647)
- 4MiB cache：[`backend/mod.rs:22-30`](../../akernel/src/distill-fs/src/backend/mod.rs#L22) [`cache.rs:385-469`](../../akernel/src/distill-fs/src/backend/cache.rs#L385)
- peer与Redis owner index：[`peer.rs:1129-1162`](../../akernel/src/distill-fs/src/backend/peer.rs#L1129) [`peer.rs:1986-2053`](../../akernel/src/distill-fs/src/backend/peer.rs#L1986)
- Dragonfly默认/启用说明：[`deploy/README.md:83-95`](../../akernel/deploy/README.md#L83)

### AgentENV

- OCI native/standard分支：[`oci_image.rs:1-30`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L1) [`oci_image.rs:198-275`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L198)
- LSMT mapping/index：[`format.rs:7-55`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/format.rs#L7) [`index.rs:627-667`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/index.rs#L627)
- ZFile random access：[`zfile.rs:720-839`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L720) [`zfile.rs:962-1040`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L962)
- registry Range：[`registryfs_v2.rs:704-876`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L704)
- remote-block cache：[`image_service.rs:278-319`](../../kvcache-ai/AgentENV/storage/overlaybd/src/image/image_service.rs#L278) [`cache_store.rs:519-636`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/cache/full_file_cache/cache_store.rs#L519)
- ublk target：[`overlaybd_target.rs:161-223`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L161)
- background download：[`bk_download.rs:215-310`](../../kvcache-ai/AgentENV/storage/overlaybd/src/bk_download.rs#L215) [`bk_download.rs:718-840`](../../kvcache-ai/AgentENV/storage/overlaybd/src/bk_download.rs#L718)
- P2P facade：[`facade.rs:246-299`](../../kvcache-ai/AgentENV/src/overlaybd/p2p/facade.rs#L246) [`p2p-design.md:90-150`](../../kvcache-ai/AgentENV/docs/src/internals/p2p-design.md#L90)
- 默认配置：[`default.toml:54-73`](../../kvcache-ai/AgentENV/config/default.toml#L54) [`default.toml:155-197`](../../kvcache-ai/AgentENV/config/default.toml#L155)

### Agent Substrate

- Golden Actor创建、warm-up与Suspend：[`actortemplate_controller.go:80-182`](../../agent-substrate/substrate/cmd/atecontroller/internal/controllers/actortemplate_controller.go#L80)
- own/golden/cold-boot选择顺序：[`workflow_resume.go:70-106`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_resume.go#L70)
- Golden external checkpoint与scope：[`workflow_suspend.go:114-176`](../../agent-substrate/substrate/cmd/ateapi/internal/controlapi/workflow_suspend.go#L114)
- image cache设计：[`imagecache/README.md:1-33`](../../agent-substrate/substrate/internal/imagecache/README.md#L1)
- pull/singleflight：[`imagecache.go:256-310`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L256) [`imagecache.go:342-437`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L342)
- 完整download/decompress/untar：[`imagecache.go:439-503`](../../agent-substrate/substrate/internal/imagecache/imagecache.go#L439)
- overlay compose：[`bundle_linux.go:35-147`](../../agent-substrate/substrate/internal/imagecache/bundle_linux.go#L35)
- microVM virtiofs接线：[`ateom-microvm/run.go:478-543`](../../agent-substrate/substrate/cmd/ateom-microvm/run.go#L478)
- scheduler无cache affinity：[`scheduling.go:89-125`](../../agent-substrate/substrate/cmd/ateapi/internal/scheduling/scheduling.go#L89)
- lazy/Preload/GC路线图：[`imagecache/README.md:158-178`](../../agent-substrate/substrate/internal/imagecache/README.md#L158)

## 15. 参考资料

1. Huiba Li, et al. *DADI: Block-Level Image Service for Agile and Elastic Application Deployment*. USENIX ATC 2020. [USENIX](https://www.usenix.org/conference/atc20/presentation/li-huiba)；[本地 PDF](<../References/Li 等 - 2020 - DADI Block-Level Image Service for Agile and Elastic Application Deployment.pdf>)。
2. Ao Wang, et al. *FaaSNet: Scalable and Fast Provisioning of Custom Serverless Container Runtimes at Alibaba Cloud Function Compute*. USENIX ATC 2021. [USENIX](https://www.usenix.org/conference/atc21/presentation/wang-ao)；[本地 PDF](<../References/Wang et al_2021_{FaaSNet}.pdf>)。
3. DragonflyOSS Nydus. [GitHub](https://github.com/dragonflyoss/nydus)；AKernel固定的 revision见本文源码索引。
4. containerd nydus-snapshotter. [GitHub](https://github.com/containerd/nydus-snapshotter)。用于理解标准OCI annotations和完整stock integration；AKernel当前不是直接运行它。
5. Alibaba Cloud OverlayBD. [GitHub](https://github.com/containerd/accelerated-container-image)。DADI/OverlayBD格式和containerd集成的上游背景；当前 AgentENV 为仓库内 Rust storage/ublk实现，不能默认视作同一版本性能。
6. Xiaohu Chai, et al. *Fork in the Road: Reflections and Optimizations for Cold Start Latency in Production Serverless Systems*. OSDI 2025. [USENIX](https://www.usenix.org/conference/osdi25/presentation/chai-xiaohu)；[本地 PDF](<../References/Chai et al_2025_Fork in the Road.pdf>)。本文引用的seed tree、FRI、EPT Prefill和性能数字来自该论文；AFaaS结果不等于AKernel实测。
