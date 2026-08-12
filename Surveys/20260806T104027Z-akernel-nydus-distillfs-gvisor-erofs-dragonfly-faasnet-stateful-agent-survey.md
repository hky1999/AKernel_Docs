# AKernel 的 Nydus、distill-fs、gVisor/EROFS 与 Dragonfly：从镜像按需加载到有状态 Agent 生命周期

> 调研时间：2026-08-06 18:40:27（Asia/Shanghai）  
> 文档时间戳：`20260806T104027Z`  
> AKernel commit：`5684c858cd835db053adef1265419af64e67bd04`  
> `sandboxd` submodule：`60e6a33e1f2b1ae04c10586c238f6d0200d0c315`  
> `distill-fs` submodule：`5f3d6d3979a4ed76d4bb6722d7c1ce6db7126634`  
> AKernel 论文 commit：`09d5247a62094291600f9b93c181082770a19852`  
> AgentENV commit：`6296bc4be7ad79eb3a278eb5264ef011c341adf5`  
> Nydus image-service 对照版本：`4345613858cc10e98149319130cb06adfb1f21e0`  
> containerd nydus-snapshotter 对照版本：`8a7cfda01d877c33b2e4816689fe4e06cad865da`  
> gVisor 对照版本：`release-20260706.0`，commit `a29997e6c74152f06a69dda686f8196d9ba7d5b2`

> 方法：固定版本源码审计、AKernel 论文草稿、Nydus/gVisor 官方源码、YuanRong、FaaSNet、DADI 论文和已有 Survey 交叉核对。本文没有在同一台服务器上完成 AKernel、AgentENV、Dragonfly、FaaSNet 的 head-to-head benchmark；所有性能判断均区分“论文实测”“源码事实”和“机制推断”。

## 1. 结论摘要

### 1.1 最重要的十个结论

1. **Nydus 不是单一 daemon，也不等于一种 FUSE。** 它至少包含 OCI 镜像约定、RAFS bootstrap/blob 格式、`nydus-image` 构建器、`nydusd` 运行时服务以及 `nydus-snapshotter` 的 containerd 集成。其核心思想是把文件系统元数据放在较小的 bootstrap 中，把文件内容切为可随机读取、可校验的压缩 chunk 放在 data blobs 中，因此容器可以在数据 blobs 尚未全部落盘时先得到文件系统视图。

2. **AKernel 消费的是标准 Nydus OCI/RAFS v5 镜像，但没有运行 stock `nydusd`。** `distill_fs` 固定依赖 Nydus 的 `nydus-api`、`nydus-rafs`、`nydus-storage` 和 `nydus-utils` crates，自行实现只读 FUSE server。准确表述应是：

   > AKernel uses a custom FUSE reader, `distill_fs`, built on Nydus libraries to consume standard Nydus OCI/RAFS v5 images.

   证据见 [`distill-fs/Cargo.toml:29-32`](../../akernel/src/distill-fs/Cargo.toml#L29)、[`image/nydus.rs:169-195`](../../akernel/src/distill-fs/src/image/nydus.rs#L169) 和 [`fs.rs:64-86`](../../akernel/src/distill-fs/src/fs.rs#L64)。

3. **AKernel 的 Nydus 与 runsc 是通过“宿主机目录”拼接的，不是 virtio-fs，也不是把 RAFS block device 传给 gVisor。** `distill_fs` 先在宿主机挂出一个 FUSE rootfs 目录；`sandboxd` 把该目录写入 OCI `spec.Root.Path`；runsc 再把它作为 gofer/LISAFS lower，并在 gVisor 内部叠加每 sandbox 私有的 memory upper。应用读文件时的路径是：

   `gVisor 应用 → gVisor VFS/overlay → gofer → host FUSE → distill_fs → RAFS chunk → registry/Dragonfly proxy`。

4. **EROFS 与上述 Nydus 路径是不同路径。** AKernel 的 Nydus reader当前明确拒绝 RAFS v6，因此不会走 Nydus v6 的 EROFS-compatible metadata。AKernel 内置的 `yr-runtime-rootfs.img` 和用户传入的 S3 raw rootfs 才是 EROFS image file；`sandboxd` 给 runsc 写 `dev.gvisor.spec.rootfs.type=erofs`，由 gVisor 自己的 EROFS 实现打开 image FD。**EROFS 是只读磁盘文件系统格式，不是内存文件系统；内存部分是 gVisor 的 tmpfs-backed writable overlay。**

5. **AKernel 当前有四条 rootfs 路径，不能合并成一句“都由 Nydus lazy load”。**

   - Nydus OCI：`distill_fs --src nydus` 挂出文件目录，按文件 chunk lazy read；
   - S3/OSS raw EROFS：`distill_fs --src oss` 挂出一个可按范围读取的 image file，gVisor 在文件内部解析 EROFS；
   - local EROFS：本地 image file 直接交给 gVisor，通常没有远端 lazy read；
   - 普通 OCI：完整 pull、逐层 extract，再挂只读 host OverlayFS 目录，是 fallback，不是 lazy load。

6. **Nydus 与 OverlayBD 都要求预转换镜像才能实现真正的 runtime lazy load。** AKernel 找不到 Nydus 原镜像或 `${image}_nydus_v3` companion 时会退化为普通 OCI 完整拉取/解包；AgentENV 面对标准 OCI tar layer 且 converted commit cache 未全命中时，也必须完整 `regctl image copy` 并转换，只有 OverlayBD-native layer 才由 `registryfs_v2` 在运行时按 Range 读取。因此不能用“支持 Nydus/OverlayBD”推导“任意 Docker image 都可立即 lazy boot”。

7. **Nydus/distill-fs 的优势是文件语义和内容寻址，OverlayBD 的优势是块设备、可写 upper 和 snapshot 统一。** Nydus 从 pathname/inode 精确找到文件 chunk，适合稀疏文件工作集、以 digest 标识内容和跨镜像 chunk dedup；OverlayBD 从 guest LBA 经 LSMT 找压缩 block，适合 Firecracker virtio-blk/ublk、ext4 兼容、可写块级 upper、rootfs/extra drive/memory snapshot 统一。AKernel 当前的 Nydus lower 只读，写入 gVisor memory upper；AgentENV 的 OverlayBD upper 可 seal/restack 成新 lower。

8. **AKernel 当前真正接通的跨节点 P2P 是可选 Dragonfly HTTP proxy，不是 `distill_fs` 自己的 peer backend。** `distill_fs` 源码已有 TCP/Unix chunk server、Redis discovery 和 checksum owner index，但 `sandboxd` 只启动 `mount` 子命令，Helm/Terraform 也没有启动 `serve-chunk`。Dragonfly 默认关闭；启用后，registry 和 OSS backend 的 proxy URL 都指向 `dragonfly-seed-client:4001`。而且 AKernel chart 明确 `client.enable=false`，因此 worker node 不是默认 Dragonfly peer；当前拓扑更接近“多副本 seed-client 代理/缓存池”。

9. **Dragonfly 与 FaaSNet 都减少 origin 流量，但系统角色不同。** Dragonfly 是通用制品 P2P/CDN；FaaSNet 把 function tree manager 嵌入 FaaS scheduler，把 worker 嵌入每个 FaaS VM agent，以“每函数一棵动态平衡二叉树”直接让 worker VM 边接收 512 KiB block、边向两个 child relay。FaaSNet 的论文数字证明这种紧耦合拓扑在预留 VM 池上的 burst scaling 很强，但不能直接外推到 AKernel Dragonfly，也不包含 microVM cold boot 时间。

10. **AKernel 与 FaaSNet 的本质差异不应只写成“FaaS 用完就删、Agent 一直运行”。** 商用 FaaS 也会缓存实例，FaaSNet 的 VM 是 15 分钟 idle 后才回收。真正差异是：FaaSNet 管理的是“函数调用所需的不可变镜像分发”，而 Agent sandbox 的逻辑身份跨多轮动作、等待、分支和恢复，状态还包括可变 workspace、进程树、匿名内存、终端/浏览器、凭证和策略。AKernel 的论文已形成这一论点，但当前开源 v0.1 只实现了 live sandbox 与 lazy rootfs；Fork、透明 Checkpoint/Restore、APlane lineage 仍是 roadmap/TODO，不能作为现有实现写入实验结论。

### 1.2 一句话回答用户最关心的问题

**当前 AKernel 的最佳定位不是“把 FaaSNet 的镜像分发换成 Nydus”，而是“用 Nydus/Dragonfly解决 Agent APlane 的不可变 lower 工作集，再用 YuanRong 式对象系统承载可寻址的可变 diff/checkpoint；最终由一个状态清单把 rootfs、进程内存、对象引用与策略绑定为可恢复的长时 Agent 环境”。后半段目前仍是设计方向，不是已完成能力。**

## 2. 范围、术语和证据边界

### 2.1 本文比较的实际对象

| 名称 | 本文实际分析的对象 | 不是 |
|---|---|---|
| 标准 Nydus | OCI annotations/media types、RAFS bootstrap/blob、nydus-image、nydusd、nydus-snapshotter | 单个 FUSE daemon 的同义词 |
| AKernel Nydus path | `sandboxd image-manager + distill_fs + RAFS v5 + runsc` | stock nydusd、RAFS v6/fscache |
| AKernel EROFS path | local/S3 raw EROFS image file + gVisor internal EROFS + memory upper | “EROFS 内存文件系统” |
| AgentENV OverlayBD | native OverlayBD/ZFile + LSMT + ublk + Firecracker | 普通 OCI 无条件 lazy boot |
| AKernel Dragonfly | 可选 Helm Dragonfly seed-client proxy，供 registry/OSS backend 使用 | 已默认部署的 per-worker dfdaemon mesh |
| FaaSNet | 论文中的 Alibaba Function Compute 集成和 Function Tree | 通用持久 Agent checkpoint 系统 |

### 2.2 三类证据

- **源码事实**：当前固定 commit 可直接验证的控制路径、参数、格式和限制。
- **论文实测**：论文指定环境中的结果，不归因到当前 AKernel 实现。
- **机制推断/设计建议**：从源码结构推导，需要后续 benchmark 或实现验证。

尤其要避免四个误读：

1. `fsManager.Restore` 只是在 `sandboxd` 重启后重建 rootfs/mount 引用，不是进程/匿名内存 C/R。[`fsmanager.go:266-351`](../../akernel/src/sandboxd/internal/server/fsmanager.go#L266)
2. `SeedInfo` 当前只有 `Cid/Cmd/Envs`，不是 live fork seed。[`manager.go:25-29`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L25)
3. gVisor `overlay2=root:memory` 不会自动把写层发布为持久 snapshot。
4. YuanRong Data System 能传对象，不等于它已经能按页恢复 gVisor 进程或按文件系统 chunk 提供 rootfs。

## 3. Nydus 到底是什么

### 3.1 把 tar layer 改成“可挂载元数据 + 可随机读取数据”

传统 OCI layer 通常是 tar/tar+gzip。容器启动前，runtime 往往需要下载 layer、解压、逐项把文件 materialize 到 snapshotter 目录；它知道 layer 中有压缩字节，却不能在不解析 tar 的情况下直接回答“`/usr/lib/libX.so` 位于哪个远端 byte range”。

Nydus image 在构建时完成这项重排：

- **bootstrap/meta blob**：superblock、inode/目录、xattr、file chunk、blob table、prefetch table等元数据；
- **data blobs**：文件内容 chunk 的压缩 payload；
- **chunk descriptor**：把某个文件区间映射到 `blob_index + compressed_offset + compressed_size + uncompressed_size + digest`；
- **OCI 封装**：data blobs 与最后一个 bootstrap layer 仍由 OCI manifest 发布和寻址。

AKernel 的检测逻辑遵循标准 Nydus annotations：

- `containerd.io/snapshot/nydus-bootstrap`
- `containerd.io/snapshot/nydus-blob`
- `application/vnd.oci.image.layer.nydus.blob.v1`
- bootstrap tar 内路径 `image/image.boot`

源码见 [`bootstrap.go:31-68`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go#L31) 和 [`bootstrap.go:71-166`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go#L71)。

### 3.2 RAFS v5 与 v6

Nydus 的 RAFS metadata 有两条重要演进路径：

| 特性 | RAFS v5 | RAFS v6 |
|---|---|---|
| metadata 布局 | Nydus 自定义 superblock/inode/chunk tables | EROFS-compatible superblock/inode，另有 Nydus 扩展表 |
| 典型解析者 | nydusd/用户态 RAFS parser | nydusd，或 Linux EROFS + fscache/block path |
| AKernel `distill_fs` | 支持 | 明确拒绝 |
| 加密/batch chunk | 当前 distill 不支持 | 标准 Nydus 完整实现可有更丰富能力 |

Nydus image-service 的 v5 chunk info保存 digest、blob index、compressed/uncompressed offset/size 和 file offset；v6 compact inode与 EROFS 兼容，详细 chunk info可放在 bootstrap 尾部的 chunk table。标准 `nydusd` 还能通过 FUSE、virtio-fs、fscache/EROFS 等不同 frontend 暴露同一个 RAFS 视图。

这里的关键不是“v6 一定更快”，而是**v6 允许内核 EROFS 接手 pathname/inode/page-cache 路径，nydusd主要按 fscache 请求回填 data blob；v5 FUSE path 则由用户态 daemon同时处理元数据和数据读取。**

### 3.3 标准 Nydus 组件边界

~~~mermaid
flowchart LR
    A[普通OCI目录/TAR] --> B[nydus-image/converter]
    B --> C[bootstrap/meta blob]
    B --> D[data blobs]
    C --> E[OCI Registry]
    D --> E

    F[Kubernetes Pod] --> G[kubelet/CRI]
    G --> H[containerd]
    H --> I[nydus-snapshotter]
    I --> J[启动/复用 nydusd]
    J --> K{frontend}
    K -->|FUSE| L[host mount目录]
    K -->|virtio-fs| M[guest filesystem]
    K -->|fscache| N[Linux EROFS]
    J --> O[BlobCache/backend]
    O --> E
~~~

- `nydus-image` 是构建/转换工具，不在每次容器 read 的 foreground path。
- `nydusd` 是 RAFS filesystem/data service，可服务多个 frontend。
- `nydus-snapshotter` 是 containerd proxy snapshotter，负责 `Prepare`、daemon 生命周期和把 remote mount返回 containerd；Kubernetes 仍通过 CRI/containerd 创建 Pod。
- AKernel 没有使用上述 snapshotter，也没有直接运行 `nydusd`，而是把所需子集嵌入 `distill_fs`。

## 4. AKernel 如何实现 Nydus lazy load

### 4.1 从 Python SDK 到 rootfs prepare

`Sandbox(image=...)` 被翻译成 openYuanRong `InvokeOptions.custom_extensions["rootfs"]`，其中 `type=image`、`readonly=false`、`imageurl=<ref>`。[`_openyuanrong.py:89-110`](../../akernel/sdk/python/akernel_sdk/_openyuanrong.py#L89) [`_openyuanrong.py:146-163`](../../akernel/sdk/python/akernel_sdk/_openyuanrong.py#L146)

节点侧 `sandboxd.Start` 同时发起两项准备：

- `fsMgr.Prepare(startReq)`：准备 rootfs 与额外挂载；
- `prepareStartResources()`：准备 cgroup、网络等运行资源。

两者并行完成后，`preparedFS.RootfsPath()` 被写入 `StartSandboxRequest.Rootfs.RootDir`，再进入 runsc adapter。[`server.go:816-903`](../../akernel/src/sandboxd/internal/server/server.go#L816) [`server.go:937-955`](../../akernel/src/sandboxd/internal/server/server.go#L937)

这说明镜像 mount 与网络/资源准备可重叠；但是 bootstrap 首次获取和 FUSE ready 仍是启动 barrier。

### 4.2 Nydus 检测、suffix 与性能悬崖

`MountOCI` 不是直接执行普通 OCI pull，而是：

1. 查 Nydus positive/negative TTL cache；
2. cache miss 时 fetch manifest/image并看最后一层 bootstrap annotation；
3. 先尝试用户原始 image ref；
4. 再尝试 `imageRef + nydus_suffix`，当前默认 `_nydus_v3`；
5. 两者都不是 Nydus才走普通 OCI overlay mount。

源码见 [`http.go:428-545`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go#L428) 和 [`sandboxd_config.toml:45-53`](../../akernel/deploy/standalone/config/sandboxd_config.toml#L45)。positive TTL 默认 1 小时、negative TTL 默认 5 分钟、总容量 1000。[`nydus_cache.go:31-44`](../../akernel/src/sandboxd/pkg/imagemanager/api/nydus_cache.go#L31)

两个边界值得在论文和 benchmark 中显式记录：

- **Nydus 已检测但 mount 失败时不会 silent fallback**，避免特殊镜像被普通 OCI 路径误处理；
- cache key是 image URL/tag 而不是强制 digest，mutable tag 在 TTL 内发生替换时可能短暂持有旧格式判断。

普通 OCI fallback明确“pull and extract OCI layers, then mount readonly overlay rootfs”，不是 lazy path。[`oci/manager.go:270-277`](../../akernel/src/sandboxd/pkg/imagemanager/oci/manager.go#L270) [`oci/manager.go:340-390`](../../akernel/src/sandboxd/pkg/imagemanager/oci/manager.go#L340)

### 4.3 bootstrap、per-image daemon 与 FUSE ready

首次 Nydus mount 时，`sandboxd`：

1. 从 OCI image最后一层提取 `image/image.boot`；
2. 读取 image config中的环境变量；
3. 为该 image建立 daemon/cache/config目录；
4. 写 Nydus backend JSON，填入 registry host/repo/auth/proxy；
5. 执行：

   `distill_fs mount --src nydus --bootstrap ... --cache-dir ... --cfg ... --chunk-db-dir ... --image-meta-dir ...`

6. 用 `statfs` 返回 zero blocks 判断 FUSE mount ready。

证据见 [`daemon.go:451-492`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L451)、[`daemon.go:643-655`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L643) 和 [`daemon.go:662-779`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go#L662)。

同一 `RootfsConfig` 通过 `rootfsMap + ready channel` 合并并发创建；`RootFS` 又维护 refcount，最后一个 sandbox 释放后才 unmount。[`manager.go:31-87`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go#L31) [`rootfs.go:181-220`](../../akernel/src/sandboxd/internal/langrtmanager/rootfs.go#L181)

因此 same-node same-image 的 N 个 sandbox：

- 共享一个 host FUSE mount、bootstrap、blob sparse cache 和 node ChunkDB；
- 每个 sandbox仍有独立 runsc/gVisor实例和独立 memory upper；
- 首次 mount 是共享 barrier，首个 caller 的 registry/bootstrap/FUSE 延迟会阻塞同批等待者。

### 4.4 `distill_fs` 内部 read path

`distill_fs` 并不是调用一个 Nydus CLI，而是直接构造：

- `GeneralBackend`：从 `BackendConfigV2` 建 registry/HTTP proxy/OSS/S3/local backend；
- `RafsSuper::load_from_file`：加载 bootstrap；
- `NydusImage`：实现 `fuse_backend_rs::FileSystem`；
- `FuseSession/FuseServer`：在 `/dev/fuse` 上运行多个 worker。

[`GeneralBackend`](../../akernel/src/distill-fs/src/backend/general.rs#L65) 的 `fetch(off, buf)` 最终调用 Nydus storage `BlobReader.read`；Nydus FUSE mount入口见 [`cli.rs:457-483`](../../akernel/src/distill-fs/src/cli.rs#L457)。

一次文件 read 的路径为：

~~~mermaid
flowchart LR
    A[应用 read path,off,len] --> B[gVisor VFS/overlay]
    B --> C[gofer/LISAFS host read]
    C --> D[Linux FUSE request]
    D --> E[distill NydusImage::read]
    E --> F[RAFS inode + chunk index]
    F --> G{decompressed ChunkDB hit?}
    G -->|yes| H[返回请求slice]
    G -->|no| I[blob id + compressed range]
    I --> J{per-blob sparse cache hit?}
    J -->|no| K[registry/Dragonfly HTTP range]
    K --> L[cache 4 MiB extent]
    J -->|yes| M[读取compressed bytes]
    L --> M
    M --> N[decompress + validate shape]
    N --> H
    N -.有界队列异步.-> O[node-level ChunkDB]
~~~

具体步骤是：

1. 根据 RAFS `chunk_size` 算出请求覆盖的 start/end chunk；
2. `inode.get_chunk_info(chunk_idx)` 得到 blob index、compressed offset/size、uncompressed size 和 chunk digest；
3. 先以 RAFS digest查全局 `ChunkDB`；
4. miss 时通过 blob reader取 compressed byte range；
5. 解压完整 RAFS chunk并返回请求 slice；
6. 完整 chunk通过容量 4096 的队列、每批最多 64 个异步写入 ChunkDB；队列满会丢 store 请求，避免阻塞 foreground read。

源码见 [`image/nydus.rs:330-376`](../../akernel/src/distill-fs/src/image/nydus.rs#L330)、[`image/nydus.rs:526-573`](../../akernel/src/distill-fs/src/image/nydus.rs#L526) 和 [`image/nydus.rs:647-693`](../../akernel/src/distill-fs/src/image/nydus.rs#L647)。

还要区分“metadata里有digest”和“foreground read重新校验digest”：当前远端 Nydus read path用 `chunk_id`作为ChunkDB key，但没有在 `read_chunk_data` 中对取回并解压的数据重新计算digest；peer ingress则会重算checksum后再写ChunkDB。[`image/nydus.rs:297-376`](../../akernel/src/distill-fs/src/image/nydus.rs#L297) [`peer.rs:2865-2873`](../../akernel/src/distill-fs/src/backend/peer.rs#L2865) 因此不能把标准Nydus完整实现的校验能力无条件归给当前 `distill_fs` foreground path。

### 4.5 两种“chunk”不能混淆

标准 Nydus 默认 RAFS chunk常见为 1 MiB，并允许更大配置；但 `distill_fs` 自己把 backend cache/dedup `CHUNK_SIZE` 固定为 4 MiB。[`backend/mod.rs:22-30`](../../akernel/src/distill-fs/src/backend/mod.rs#L22)

当前实际有两层粒度：

- **RAFS file chunk**：由 bootstrap metadata定义；用于文件区间映射、压缩和 Nydus digest；
- **distill backend extent**：4 MiB；per-blob sparse cache的 bitmap位和 raw backend DedupReader以此为单位。

由于 AKernel Nydus daemon总是传非空 `--cache-dir`，一次很小的 compressed chunk miss可能触发其所在 4 MiB blob extent 的远端读取。优点是顺序或局部后续读取命中，代价是稀疏小读可能产生 read amplification。标准 nydusd 的 BlobCache可直接围绕 `BlobChunkInfo` 合并连续 chunk range并 prefetch；当前 `distill_fs` 的 FUSE loop逐 chunk读取，没有使用完整 `BlobDevice/BlobIoVec` 合并路径。这是值得 benchmark 的实现差异，而不是 Nydus 格式本身的必然限制。

### 4.6 当前支持边界

当前 `distill_fs`：

- 只读；
- Linux/FUSE only；
- RAFS v5 only；
- RAFS v6返回 `Unsupported`；
- encrypted chunk不支持；
- batched chunk不支持；
- backend可为 local、OSS、S3、registry、HTTP proxy；
- 有 per-image sparse blob cache、node-level LMDB ChunkDB/IndexDB 和可选 peer代码。

详见 [`distill-fs/README.md:1-59`](../../akernel/src/distill-fs/README.md#L1) 与 [`image/nydus.rs:169-205`](../../akernel/src/distill-fs/src/image/nydus.rs#L169)。所以它是“基于 Nydus 的专用读取器”，不是 Nydus 全功能 runtime 的 drop-in replacement。

## 5. `distill_fs`、runsc、EROFS 和 memory overlay 的准确关系

### 5.1 AKernel 当前四条 rootfs 路径

~~~mermaid
flowchart TB
    A[AKernel rootfs input] --> B{来源/格式}

    B -->|Nydus OCI| C[提取RAFS v5 bootstrap]
    C --> D[distill_fs FUSE目录]
    D --> E[runsc: directory rootfs]
    E --> F[gVisor gofer lower + memory upper]

    B -->|S3/OSS raw image| G[distill_fs FUSE单文件]
    G --> H[regular EROFS image file]
    H --> I[runsc rootfs annotation type=erofs]
    I --> J[gVisor EROFS lower + memory upper]

    B -->|local yr-runtime-rootfs.img| K[本地 EROFS file]
    K --> I

    B -->|普通 OCI| L[完整pull/extract]
    L --> M[host readonly OverlayFS目录]
    M --> E
~~~

`distill_fs` raw模式只在 mountpoint下暴露一个 regular file；`MountOSS` 返回的 `FilePath` 是 `mountpoint/name`。[`image/raw.rs:1-35`](../../akernel/src/distill-fs/src/image/raw.rs#L1) [`http.go:133-216`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go#L133)

OCI loader检查 `Root.Path`：

- 是目录：不写 gVisor rootfs-image annotations；
- 是 regular file：创建 bundle placeholder，写 `source=<abs path>`、`type=erofs`，可写 root时再写 `overlay=memory`。

源码见 [`oci_loader.go:183-225`](../../akernel/src/sandboxd/pkg/runtime/oci_loader.go#L183)。runsc命令同时显式使用 `--overlay2=root:memory[,size=...]`。[`runsc/client.go:107-145`](../../akernel/src/sandboxd/pkg/runtime/runsc/client.go#L107)

### 5.2 EROFS 不是“内存文件系统”

应把三个概念拆开：

1. **EROFS lower**：由 `mkfs.erofs` 生成的只读磁盘 image format；
2. **gVisor EROFS implementation**：runsc从 annotation拿到 image file FD，gVisor Sentry调用 `erofs.OpenImage`并把它注册为 root filesystem；这不是宿主 Linux 内核 mount EROFS；
3. **memory upper**：gVisor overlay 的 tmpfs upper；所有写入、删除和 copy-up发生在此。

AKernel 构建默认 runtime image时执行：

`mkfs.erofs -E noinline_data /yr-runtime-rootfs.img /rootfs`

见 [`builder/runtime.Dockerfile:215-228`](../../akernel/builder/runtime.Dockerfile#L215)。gVisor对应源码中：

- rootfs annotations prefix是 `dev.gvisor.spec.rootfs.`；
- `type=erofs` 分支把 image FD交给 gVisor EROFS；
- overlay启用时强制 lower read-only，创建 tmpfs upper，注释明确“keep all modifications inside the sandbox”。

因此：

> EROFS 决定 lower 如何组织和读取；memory overlay决定 sandbox写到哪里。它们是正交的 lower/upper，不是同一个文件系统。

### 5.3 EROFS 与 Nydus 的关系

标准 Nydus RAFS v6 的 metadata布局与 EROFS兼容，使 Linux EROFS可解析 inode/目录并通过 fscache向 nydusd请求缺失 data blob。那条路径可表示为：

`application → Linux EROFS/page cache → fscache on-demand request → nydusd BlobCache → remote blob`。

AKernel 当前 Nydus路径却是：

`application → gVisor/gofer → host FUSE → distill_fs RAFS v5 parser → remote blob`。

因为 `distill_fs` 在加载到 v6时立即返回 unsupported，不能以“AKernel Nydus使用 EROFS加速”描述当前代码。[`image/nydus.rs:185-195`](../../akernel/src/distill-fs/src/image/nydus.rs#L185)

### 5.4 可写性和持久性的现实边界

Nydus/EROFS lower可被很多 sandbox共享，但每个 sandbox的 write set在 gVisor memory upper中：

- sandbox运行时可以正常写 `/tmp`、workspace和安装包；
- 写入消耗 sandbox内存/受 tmpfs size限制；
- 删除 sandbox时 upper不因 Nydus 自动持久化；
- 另一个 sandbox共享的是同一 immutable lower，不共享前者 private upper；
- 当前没有“seal upper → 发布 Nydus delta → 形成新 APlane version”的代码。

这对短 benchmark可能无碍，却是长时 Agent的核心问题：代码修改、构建缓存和任务产物往往比首次镜像工作集更重要。

## 6. Nydus lazy load 与 AgentENV OverlayBD lazy load

更完整的 OverlayFS/DADI/OverlayBD基础解释见 [DADI、OverlayFS 与 OverlayBD Survey](./20260806T070035Z-dadi-overlayfs-overlaybd-block-layer-lazy-pulling-explained-survey.md)，AKernel/AgentENV/Substrate对比见 [镜像路径对比 Survey](./20260805T073718Z-akernel-nydus-agentenv-overlaybd-substrate-rl-rollout-image-lazy-load-survey.md)。本节只收敛与本题直接相关的差异。

### 6.1 同构思想：先拿索引，访问时才取 payload

二者共同采用：

1. 镜像发布前重排数据，使内容可随机访问；
2. 启动时只取得 metadata/index；
3. 文件或 block首次访问时计算 remote range；
4. 用 checksum/CRC验证并解压；
5. 缓存已经取回的数据；
6. 允许后台下载/P2P把 runtime miss逐渐转为本地 hit。

差异是“谁发起访问”和“索引表达什么”。

### 6.2 端到端映射对比

| 维度 | AKernel Nydus/`distill_fs` | AgentENV OverlayBD |
|---|---|---|
| 运行时入口 | host FUSE directory | `/dev/ublkbN` block device |
| 请求语义 | pathname/inode/file offset | guest logical block address |
| metadata | RAFS bootstrap：inode/dir/chunk/blob table | LSMT header/index + ZFile jump table |
| lazy单位 | RAFS file chunk；distill blob cache另以4 MiB extent缓存 | compressed block/range，guest看逻辑sector |
| 典型路径 | gVisor gofer → FUSE → RAFS → blob range | Firecracker virtio-blk → ublk → LSMT → ZFile → registry range |
| lower文件系统 | RAFS本身提供文件视图 | guest通常在虚拟块设备上挂ext4 |
| 可写层 | gVisor memory tmpfs upper，当前不进入 Nydus lineage | LSMT writable upper，可seal/restack为lower |
| snapshot亲和性 | rootfs read-only；进程/内存C/R需另做 | rootfs/drive/memory snapshot都可复用OverlayBD/ublk路径 |
| 跨镜像去重 | RAFS digest与ChunkDB天然是content-aware | 主要依赖layer/block cache和相同commit复用 |
| 格式兼容 | 当前只RAFS v5子集 | OverlayBD-native/ZFile；标准OCI需转换 |

AgentENV native path只生成 `repoBlobUrl + digest + size` 的 remote `image.json`，由 `registryfs_v2`按 Range读取。[`oci_image.rs:1-30`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L1) [`oci_image.rs:198-235`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L198) ublk收到 read/write/flush/discard后调用 OverlayBD `ImageFile`。[`overlaybd_target.rs:310-388`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L310)

### 6.3 Nydus/AKernel 更可能占优的场景

- 应用只访问镜像中少数已知文件；
- 大镜像由大量很少触达的 Python/浏览器/toolchain依赖组成；
- 跨镜像有相同文件 chunk，可用 checksum ChunkDB复用；
- 希望以文件路径定义 prefetch、热点和安全审计；
- runsc/gVisor本来就消费 host directory rootfs，不需要给 microVM造块设备。

代价包括：FUSE + gofer的额外请求边界、当前逐 RAFS chunk读取、4 MiB backend extent读放大、v5-only、没有持久 writable upper。

### 6.4 OverlayBD/AgentENV 更可能占优的场景

- Firecracker需要标准 virtio-blk root disk；
- guest文件系统、分区、inode等无需宿主理解；
- 要支持任意块级写、discard、extra drive；
- pause/snapshot时希望 rootfs delta和 memory image共享同一 layered storage；
- 需要把 live writable upper快速seal/restack，不想重新构建文件级镜像。

代价包括：block layer不知道哪个路径重要，可能读取 ext4 metadata/indirect blocks；内容级跨镜像 dedup较弱；普通 OCI 首次仍需完整转换；guest filesystem与LSMT/ZFile/ublk链路更复杂。

### 6.5 对 Agent workload 的真正选择标准

不应问“文件级一定比块级快吗”，而应测：

- `TTUA`：第一条有效 Agent动作，而不是 create API返回；
- touched bytes / image bytes；
- origin、P2P和node-local流量；
- p95/p99 first-touch stall；
- FUSE/gofer或ublk/VM-exit CPU；
- cache占用与跨sandbox复用；
- write set大小、snapshot publish成本和restore工作集。

顺序扫描几乎整个镜像时，lazy优势会收敛，额外索引/FUSE/Range/解压可能成为负担；稀疏工作集和高复用 fan-out才是两类方案的主战场。

## 7. AKernel 的 Dragonfly 与 FaaSNet

### 7.1 AKernel 当前有两套容易混淆的 P2P 概念

#### A. `distill_fs` peer：源码存在，部署未接通

`distill_fs serve-chunk` 可以同时提供：

- 本机 Unix socket给 mount daemon；
- TCP chunk server给其他节点；
- static peers或 Redis peer discovery；
- Redis checksum→owner index；
- peer健康、RTT、circuit breaker和 whole-chunk checksum验证。

例如 DedupReader本地 miss时可请求 local chunk server触发peer prefetch，再回查 ChunkDB。[`dedup.rs:517-550`](../../akernel/src/distill-fs/src/backend/dedup.rs#L517) peer端拿到数据后重算 checksum才写入本地 ChunkDB。[`peer.rs:2855-2873`](../../akernel/src/distill-fs/src/backend/peer.rs#L2855)

但是当前 `sandboxd` mount args没有 `--chunk-server-sock`、peer discovery或`serve-chunk`进程，AKernel部署文件中也没有启动该服务。因此它应标注为“distill-fs已有能力、AKernel尚未接线”。

#### B. Dragonfly：部署已提供但默认关闭

`install_dragonfly=false` 是默认值；启用时Terraform安装官方 chart `1.6.21`，默认创建3个seed node和1个server node pool。[`variables.tf:630-706`](../../akernel/deploy/terraform/aliyun/variables.tf#L630)

AKernel把同一个 seed-client HTTP地址同时注入：

- OSS backend proxy；
- Nydus registry backend proxy；
- bootstrap/image fetch proxy。

[`values-akernel.yaml.tmpl:131-164`](../../akernel/deploy/terraform/aliyun/values-akernel.yaml.tmpl#L131) `sandboxd` 又从 Nydus backend template提取 proxy URL给 bootstrap fetch。[`manager.go:431-439`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/manager.go#L431)

但 Dragonfly values明确：

- `seedClient.replicas = seed_replicas`；
- seed client有持久卷；
- `client.enable=false`。

见 [`values-dragonfly.yaml.tmpl:70-123`](../../akernel/deploy/terraform/aliyun/values-dragonfly.yaml.tmpl#L70)。这意味着默认 AKernel compute worker不运行 Dragonfly client/dfdaemon，也不把刚读到的块直接向其他 compute worker relay。数据先进入 seed-client pool，再由各 worker的 `distill_fs`本地 cache复用。

### 7.2 FaaSNet 的设计

FaaSNet不是外置通用代理，而是 FaaS平台内生的数据面：

- 每个 function有独立 Function Tree；
- tree manager嵌入 FaaS scheduler；
- worker嵌入每个 FaaS VM agent；
- tree是动态平衡二叉树，每个中间 VM一个 parent、最多两个children；
- root从registry取数据，中间节点收到完整 block后立即发给children；
- image在部署阶段转为带 offset table的独立压缩 block格式；
- 默认 block size为512 KiB；
- worker使用非阻塞 `sendmsg/recvmsg`、pipeline和out-of-order receive；
- VM idle 15分钟后可被回收，tree执行delete/rebalance；
- scheduler的 FT metadata在内存，周期同步到etcd。

这把放置、扩缩、拓扑和镜像streaming联合优化，避免所有新 VM同时读registry，也避免单个root承担复杂layer-tree协调。

### 7.3 论文性能数字及其适用边界

FaaSNet ATC'21论文报告：

- 128并发：baseline平均83.3 s，Kraken 100.4 s；
- FaaSNet比baseline快13.4×、比Kraken快16.3×、比registry on-demand快5×、比DADI+P2P快2.8×；
- 128并发时首个函数5.5 s、最后一个7 s，启动展开区间1.5 s；DADI+P2P完成128个需22.3 s；
- 1000 VM、2500 function在5.1–8.3 s完成container provisioning；
- PyStan image在512 KiB block下，相比regular docker pull网络I/O减少83.9%；
- 中间 VM聚合峰值约45 MB/s，为1 Gbps带宽的35.2%。

必须同时保留实验边界：论文使用预留 free VM pool，**container provisioning latency不含 VM cold start**；每VM 2 CPU、4 GiB、1 Gbps；主要函数运行约2 s。上述结果不能直接成为 AKernel+Dragonfly 的性能数字。

### 7.4 Dragonfly 与 FaaSNet 对比

| 维度 | AKernel 当前 Dragonfly | FaaSNet |
|---|---|---|
| 目标 | 通用文件/OCI/模型等制品分发 | FaaS custom container/code package burst startup |
| 控制面 | Dragonfly manager/scheduler，独立于AKernel调度器 | FT manager嵌入 FaaS scheduler |
| data peer | 当前主要是专用 seed-client pods | 直接是承载函数的worker VMs |
| 拓扑 | Dragonfly动态peer scheduling；AKernel暴露一个seed-client Service | 每函数一棵动态平衡二叉树 |
| 数据单位 | 通用HTTP task/piece/range，Nydus仍决定文件chunk | 自有512 KiB独立压缩block+offset table |
| streaming | 由Dragonfly任务/peer完成，compute worker当前不relay | VM收到完整block立即向children relay |
| 与placement关系 | 当前AKernel scheduler未见Dragonfly cache-aware placement实现 | function placement和FT membership直接耦合 |
| 内容范围 | immutable registry/OSS payload | immutable function image/code package |
| mutable workspace/process state | 不负责 | 不负责 |
| 集成成本 | 透明HTTP proxy，通用、低侵入 | 需要修改scheduler、VM agent和image format |
| 弹性适配 | 通用peer健康/调度；AKernel专用seed pool有独立容量 | VM join/leave触发insert/delete/rebalance |

机制上，FaaSNet更容易在“同一函数从1扩到N”时获得理想树形广播；Dragonfly更通用，能跨制品、跨应用复用缓存并处理长期节点池。AKernel当前只启用seed-client、未启用per-worker client，会让路径更可控，却牺牲利用worker出口带宽的机会；大规模全冷 burst仍可能在 seed-client Service和专用seed pool形成热点。

### 7.5 对 AKernel 的直接建议

1. 先明确是保留“专用seed pool”还是启用 per-worker Dragonfly client；两者隔离性、成本和扩散速度不同。
2. 将 Nydus bootstrap、blob range、distill decompressed chunk分别计量，避免把三者都统计成“image pull”。
3. 评估是否接通 `distill_fs serve-chunk`。它能传播 checksum chunk，但与 Dragonfly传播remote bytes可能重复缓存和重复网络；需要选定层级。
4. 把 image/chunk provider位置反馈给 AKernel scheduler，否则数据面有cache，控制面却仍随机把新 sandbox放到cold node。
5. 避免一项内容同时在 Dragonfly、distill sparse cache、ChunkDB和YuanRong Data System保存四份而没有统一GC与ownership。

## 8. FaaS 与长时、有状态 Agent sandbox 的本质差异

### 8.1 先修正一个过度简化

“FaaS运行完就销毁”不是严格事实。生产FaaS通常缓存container/microVM以服务后续invoke；FaaSNet也让VM在15分钟idle后才被scheduler回收。更准确的区别是：

- FaaS的**编程、扩缩和计费单元**主要是一次invoke；跨invoke状态通常外置；
- Agent的**逻辑会话单元**跨模型思考、工具调用、外部等待、人类反馈、分支与恢复，sandbox内状态本身就是任务的一部分。

AKernel论文 background已明确：Agent环境不能在每次函数返回后丢弃，也不能始终视为固定资源的长期服务；RL rollout是同一状态谱系上的派生，而不是完全独立冷启动。[`2-background.tex:46-76`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/2-background.tex#L46)

### 8.2 四类状态决定系统边界

| 状态 | 示例 | 能否重新获取 | Nydus/Dragonfly/FaaSNet是否解决 |
|---|---|---|---|
| immutable input | base image、toolchain、dataset、依赖 | 可以，适合共享/lazy/P2P | 是，三者主要解决此层 |
| mutable persistent | git diff、workspace、build cache、日志、产物 | 不可简单丢弃 | 否；需要upper/delta/object persistence |
| volatile runtime | 进程树、匿名内存、PTY、浏览器、tool server | 需重放或runtime checkpoint | 否；镜像P2P不包含它 |
| attached policy/state | credential handle、网络出口、配额、ACL、审计上下文 | 恢复时必须重新验证 | 否；不能无条件复制 |

这一分类来自论文 [`2-background.tex:78-93`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/2-background.tex#L78)。它揭示了 FaaSNet和Agent infrastructure的分界：**FaaSNet把immutable image送到足够多的runtime；Agent system还必须说明一个会话的可变、易失和策略状态如何等待、迁移、fork、恢复和提交。**

### 8.3 时间尺度改变了最优策略

FaaS冷启动优化通常最关心“几十毫秒到几秒后开始一次短invoke”。Agent session可能持续几十分钟或数小时，并在active和waiting之间反复切换：

- 第一次cold start只发生一次；
- workspace持续变大；
- 安装依赖和构建缓存可能超过base image工作集；
- 等待LLM/外部服务时CPU可让出，但匿名内存不能凭空丢弃；
- branch/rollback要求版本谱系，而不是重新从image启动；
- 恢复时“第一条有效动作”可能需要上次修改后的文件和live tool service。

因此，Agent的长期指标应是：

- 每个成功task的resource-hours；
- active/wait占比；
- checkpoint/resume次数及尾延迟；
- mutable bytes、anonymous memory和external handles；
- 重复setup/下载/编译量；
- branch共享比例和失败分支回收成本；
- end-to-end task success/goodput，而非只有cold start。

### 8.4 当前 AKernel 已实现与未实现

**已实现：**

- openYuanRong `@yr.instance` 作为单 sandbox actor/RPC handle；
- runsc/gVisor sandbox；
- 命令、文件、PTY、port/tunnel；
- Nydus/raw EROFS lazy rootfs和普通 OCI fallback；
- node-local mount/cache共享；
- 可选 Dragonfly proxy部署。

`_SandboxInstance` 当前持有的应用级状态只是 `cwd` 和 `_procs` map。[`_instance.py:20-40`](../../akernel/sdk/python/akernel_sdk/_instance.py#L20)

**未在开源 v0.1 实现：**

- gVisor live fork；
- transparent process/memory checkpoint/restore；
- rootfs writable upper持久化和lineage；
- APlane manifest与跨节点restore；
- checkpoint/cache locality-aware scheduler；
- policy-carrying restore。

AKernel README明确把 fork-based launch与Checkpoint/Restore标为planned/unavailable。[`README.md:33-39`](../../akernel/README.md#L33) [`README.md:165-170`](../../akernel/README.md#L165) 论文设计章的 APlane、Checkpoint、Restore、Fork/COW也仍带TODO。[`5-design.tex:72-103`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex#L72)

## 9. 如何用 YuanRong Data System 构造“带着 rootfs 的 C/R”

### 9.1 YuanRong 提供的可复用基础

YuanRong SIGCOMM'24论文中的 data system有以下性质：

- 每节点 data worker与本地函数通过共享内存交互；
- object directory记录 UUID→providers；
- 远端 data worker透明传输并缓存 object copy；
- 任一缓存副本可作为provider；
- pass-by-reference，允许异步prefetch并与计算重叠；
- 支持 immutable/mutable object、PRAM/causal consistency；
- cold copy可淘汰，primary可spill到local SSD；
- 可选 write-through到 OBS提供可靠存储；
- function private state可由应用 `SaveState/LoadState`显式保存和恢复。

论文报告 object exchange最低约200 μs，并将其定位为分布式共享内存池；这些能力非常适合存放内容寻址chunk、checkpoint objects和manifest metadata。

但当前 AKernel Python SDK初始化明确设置 `bypass_datasystem=True`。[`_openyuanrong.py:65-75`](../../akernel/sdk/python/akernel_sdk/_openyuanrong.py#L65) 所以“AKernel已经用YuanRong Data System传播rootfs/checkpoint”不是当前源码事实。

### 9.2 不要复制“一个巨大 checkpoint 文件”，而要发布状态清单

建议把一个可恢复 Agent环境建模为以下 manifest：

~~~text
APlaneVersion {
  id, parent_version, created_at,
  immutable_rootfs: {
    nydus_manifest_digest,
    bootstrap_object,
    blob_objects[]
  },
  mutable_fs: {
    upper_manifest,
    dirty_chunks[], whiteouts[], metadata_delta
  },
  runtime_checkpoint: {
    runtime_type, runtime_version,
    process_state_objects[], memory_objects[],
    cpu_features, kernel_abi, device_constraints
  },
  attached_state: {
    credential_handles[], network_policy_ref,
    quota_ref, audit_context
  },
  publication_state: PREPARING | COMMITTED
}
~~~

这里 immutable rootfs只存content-addressed引用，N个branch共享；mutable upper和memory只保存dirty增量；策略只保存可重新验证的handle，不复制长期secret。

### 9.3 建议的数据路径

~~~mermaid
flowchart LR
    A[Freeze AProc] --> B[Seal writable upper]
    A --> C[Capture process/memory]
    B --> D[chunk + hash + publish to Data System]
    C --> D
    D --> E[all objects durable/replicated]
    E --> F[commit APlane manifest]

    F --> G[Scheduler chooses target]
    G --> H[fetch tiny manifest + bootstrap]
    H --> I[mount lower immediately]
    I --> J[distill backend resolves chunk UUID]
    J --> K{local provider/cache?}
    K -->|yes| L[shared memory/local SSD]
    K -->|no| M[remote data worker/OBS]
    L --> N[restore useful working set]
    M --> N
    N --> O[restore runtime + revalidate policy]
    O --> P[first useful action]
~~~

发布顺序必须是：

1. freeze或建立一致性点；
2. 保存dirty filesystem和runtime state objects；
3. 验证object持久性/副本；
4. 最后原子发布 committed manifest。

否则调度器可能看到manifest，却拿不到其中某个memory或upper chunk。

### 9.4 Data System 本身不自动等于 lazy filesystem

YuanRong object `get` 的按需语义主要是“何时解析/传输一个object”；如果把20 GiB rootfs或checkpoint作为单object，第一次get仍可能传完整对象。要获得文件/页级lazy read，需要补一层：

- 按 RAFS chunk、filesystem delta chunk、memory extent切成独立objects；或
- Data System提供range/partial object接口；
- `distill_fs`增加 YuanRong backend：`blob/chunk id → ObjectRef → local/remote provider`；
- object directory暴露location/size/hotness给 scheduler；
- restore先取manifest/bootstrap/working set，其余按first-touch读取。

所以论文应写成：

> YuanRong Data System provides the distributed object naming, placement, transfer, caching and durability substrate; Nydus/distill-fs provides filesystem-granular interpretation and lazy faulting.

而不是：

> Putting a rootfs into YuanRong automatically makes checkpoint/restore lazy.

### 9.5 “带着 rootfs 的 C/R”如何准确表述

进程checkpoint不能脱离文件系统版本：恢复的进程可能持有某文件descriptor、mmap inode、当前工作目录或已删除但仍打开的文件。正确做法不是把base image复制进checkpoint，而是绑定：

`immutable lower digest + mutable upper version + runtime checkpoint + compatibility + policy`。

Restore时：

1. 验证CPU特性、gVisor版本、checkpoint ABI；
2. 重建完全相同的 lower/upper namespace；
3. 保证open inode与mount identity可重连；
4. lazy取回必需file/memory working set；
5. 恢复进程；
6. 重建不可迁移的网络连接/设备；
7. 重新验证credential和egress policy。

这才是“rootfs-aware C/R”。单独复制Nydus bootstrap不是C/R，单独保存runsc state也不足以保证文件版本一致。

### 9.6 YuanRong 原生 state suspend 与透明 C/R 的差异

YuanRong论文中的 idle→suspended可调用应用定义的 `SaveState`，恢复时创建新instance并调用 `LoadState`。这适合可序列化actor state，但不等价于透明保存：

- 任意子进程；
- 匿名内存；
- PTY和浏览器；
- mmap/file descriptor；
- 内核对象和网络连接。

AKernel可以同时保留两种等级：

- **semantic checkpoint**：应用显式SaveState，体积小、跨版本更稳；
- **runtime checkpoint**：gVisor/process级透明C/R，兼容要求高、状态更完整。

调度器按等待时间、checkpoint大小、重放成本和SLO选择，而不是所有 idle sandbox统一全量快照。

## 10. 面向论文的比较框架

### 10.1 不要只按“容器启动系统 vs Agent系统”分类

建议用两个轴组织AKernel与FaaSNet：

**轴一：状态覆盖范围**

`immutable image → mutable filesystem → process/memory → external policy/side effects`

FaaSNet主要覆盖第一项；当前AKernel实现覆盖第一项和live期间的private upper；AKernel论文目标覆盖全部，但后几项尚待实现。

**轴二：生命周期操作**

`cold spawn → repeated invoke → wait → checkpoint → resume → fork → migrate → commit/abort`

FaaSNet优化cold scale-out与repeated invoke所需image availability；Agent-native系统的差异化价值在wait、resume、fork、migrate和commit/abort。

### 10.2 可写进论文的核心主张

> FaaS image systems optimize the replication of immutable execution inputs for short invocation-driven scaling. Agent infrastructures must instead manage a versioned execution state whose immutable image is only one component; mutable workspace, volatile runtime state and attached policy survive across actions and may branch, wait, migrate and resume.

对应中文：

> FaaS镜像系统解决的是“如何把相同的不可变执行输入快速送到新增实例”；Agent基础设施解决的是“如何让一个会分支、等待和迁移的版本化执行状态持续存在”，镜像只是该状态的共享lower。

### 10.3 AKernel 应突出、也必须完成的系统点

1. **APlane manifest**：统一rootfs lower、upper、memory、objects和policy lineage。
2. **state-aware scheduler**：计算资源与image/checkpoint/object provider联合放置。
3. **checkpoint-on-idle**：按状态成本释放CPU/内存，不是固定15分钟删实例。
4. **rootfs-aware remote fork**：共享immutable lower，只传dirty upper/memory working set。
5. **TTUA定义**：恢复后能执行下一条有效tool action，而非process刚start。
6. **外部副作用语义**：本地rollback不能撤销已提交数据库/API副作用。

## 11. 建议实验设计

### 11.1 镜像 lazy load microbenchmark

**变量：**

- format：普通OCI、Nydus v5、raw EROFS over distill、OverlayBD-native；
- distribution：direct registry、AKernel Dragonfly seed-client、未来per-worker client、warm node cache；
- concurrency：1/8/32/128/1000 sandbox；
- nodes：1/8/32/100；
- workload：Python coding、浏览器、编译toolchain、顺序扫描；
- cache：all-cold、bootstrap-only warm、partial-working-set warm、fully warm。

**指标：**

- create API、mount ready、runsc started、first useful action四个时间点；
- p50/p95/p99；
- bootstrap bytes、blob bytes、decompressed bytes、origin bytes、P2P bytes；
- touched/image ratio；
- FUSE ops、gofer CPU、decompression CPU、cache hit；
- per-node磁盘/内存cache占用；
- seed-client/scheduler/registry峰值带宽。

必须把普通 OCI fallback和预转换 native image分开报告，否则测到的是conversion pipeline而不是lazy read。

### 11.2 Dragonfly vs FaaSNet机制复现实验

不能直接部署原FaaSNet代码时，可实现三种拓扑对比：

1. central/seed-client proxy；
2. Dragonfly per-worker peers；
3. function/image-specific balanced relay tree。

固定相同block/chunk format和origin，再比较：first/last ready spread、origin bytes、每peer入/出带宽、拓扑维护、node churn和failure recovery。这样才能把“数据格式收益”与“拓扑收益”分开。

### 11.3 长时 Agent session 实验

构造30–120分钟会话：

- 多轮命令和LLM等待；
- 修改代码、安装依赖、产生1–10 GiB workspace/cache；
- 保留background tool server/PTY；
- 从共同状态fork 8/32/128 branches；
- 跨节点resume并只访问小工作集。

比较：

- 每次从base image重建；
- 只保存rootfs diff；
- semantic SaveState；
- 完整rootfs+runtime checkpoint；
- APlane manifest + lazy working-set restore。

指标应包含resource-hours、总传输、checkpoint bytes、restore TTUA、branch amplification、任务成功率和external side-effect correctness。

### 11.4 关键消融

- no lazy load；
- no Dragonfly；
- no ChunkDB dedup；
- no locality-aware scheduling；
- memory upper vs persistent upper；
- full checkpoint vs working-set lazy restore；
- object-granular vs chunk-granular Data System backend。

## 12. 风险和待实现项

### 12.1 当前源码风险

1. **mutable tag格式缓存**：Nydus detection cache按URL/tag，有TTL stale窗口。
2. **FUSE层数**：gVisor gofer叠host FUSE，small-file metadata workload可能有高请求放大。
3. **4 MiB extent**：小随机read可能显著放大registry流量。
4. **per-image first-mount barrier**：同批sandbox可能被一个bootstrap/mount拖尾。
5. **memory upper持久性**：长时workspace占内存且删除即失；当前不是APlane。
6. **P2P重复层**：Dragonfly bytes、distill blob cache、ChunkDB、未来YuanRong object cache可能重复。
7. **seed-client热点**：worker client默认关闭，专用pool容量需要按all-cold burst设计。
8. **v5-only**：不能利用标准Nydus v6/EROFS/fscache完整路径。

### 12.2 论文边界

- 不要把 README中40 ms cold start当当前开源实测；它明确标注planned。
- 不要把 `fsManager.Restore`写成sandbox C/R。
- 不要把 `@yr.instance` actor state当透明process checkpoint。
- 不要把YuanRong论文中的Data System能力直接归为AKernel已接线；SDK当前bypass。
- 不要把Dragonfly chart安装等同于per-worker P2P。
- 不要把FaaSNet预留VM实验的绝对秒数与AKernel从零microVM/sandbox启动直接比较。

## 13. 最终判断

AKernel当前的镜像设计已经有一条清楚、可用的路径：**Nydus/RAFS v5负责文件级远端索引，`distill_fs`负责FUSE、缓存和去重，runsc/gVisor负责隔离和私有memory upper，Dragonfly可选地降低registry/OSS origin压力。** 这条路径适合大镜像、小工作集、same-node高复用的Agent sandbox。

它与AgentENV OverlayBD的主要区别不是“一个lazy、一个不lazy”，而是**文件系统lower vs块设备lower、memory upper vs可seal块upper、独立rootfs优化 vs rootfs/drive/memory snapshot统一**。二者都只有在native/preconverted格式上才能避免首次完整转换。

与FaaSNet相比，AKernel真正有论文价值的方向也不只是换一个P2P：FaaSNet已经非常有效地解决了短函数burst时immutable image的树形传播；AKernel需要证明的是，当执行单位变成长时、有状态、会等待和分支的Agent时，系统如何把immutable rootfs、mutable workspace、runtime checkpoint和policy组织为可寻址状态谱系，并通过YuanRong式数据系统完成跨节点lazy流转。

但要把这一论点坐实，下一步必须补齐：persistent upper/APlane manifest、runtime C/R、Data System chunk backend、state/cache-aware scheduler，以及以first useful action和整个session resource-hours为中心的实验。当前源码已经证明了镜像lower机制，尚未证明完整的“带rootfs有状态Agent远程C/R”。

## 14. 源码索引与参考资料

### 14.1 AKernel 本地源码

- [`sdk/python/akernel_sdk/_openyuanrong.py`](../../akernel/sdk/python/akernel_sdk/_openyuanrong.py)：rootfs请求、YuanRong初始化与`bypass_datasystem`。
- [`sdk/python/akernel_sdk/_instance.py`](../../akernel/sdk/python/akernel_sdk/_instance.py)：当前`@yr.instance` sandbox actor状态和RPC。
- [`sandboxd/internal/server/server.go`](../../akernel/src/sandboxd/internal/server/server.go)：FS/资源并行prepare与runtime启动。
- [`sandboxd/internal/langrtmanager/manager.go`](../../akernel/src/sandboxd/internal/langrtmanager/manager.go)：rootfs singleflight共享。
- [`sandboxd/internal/langrtmanager/rootfs.go`](../../akernel/src/sandboxd/internal/langrtmanager/rootfs.go)：IMAGE/S3/LOCAL mount与refcount。
- [`sandboxd/pkg/imagemanager/api/http.go`](../../akernel/src/sandboxd/pkg/imagemanager/api/http.go)：Nydus detection、suffix、fallback、OSS raw path。
- [`sandboxd/pkg/imagemanager/nydus/bootstrap.go`](../../akernel/src/sandboxd/pkg/imagemanager/nydus/bootstrap.go)：标准annotation和bootstrap提取。
- [`sandboxd/pkg/imagemanager/distillfs/daemon.go`](../../akernel/src/sandboxd/pkg/imagemanager/distillfs/daemon.go)：distill进程参数、mount ready、生命周期。
- [`sandboxd/pkg/runtime/oci_loader.go`](../../akernel/src/sandboxd/pkg/runtime/oci_loader.go)：directory rootfs与EROFS image annotation分支。
- [`sandboxd/pkg/runtime/runsc/client.go`](../../akernel/src/sandboxd/pkg/runtime/runsc/client.go)：`--overlay2=root:memory`。
- [`distill-fs/src/image/nydus.rs`](../../akernel/src/distill-fs/src/image/nydus.rs)：RAFS v5 FUSE read、解压、ChunkDB。
- [`distill-fs/src/backend/cache.rs`](../../akernel/src/distill-fs/src/backend/cache.rs)：4 MiB sparse cache/bitmap/in-flight merge。
- [`distill-fs/src/backend/dedup.rs`](../../akernel/src/distill-fs/src/backend/dedup.rs)：IndexDB/ChunkDB dedup和local peer prefetch。
- [`distill-fs/src/backend/peer.rs`](../../akernel/src/distill-fs/src/backend/peer.rs)：chunk server、discovery、index和peer protocol。
- [`builder/runtime.Dockerfile`](../../akernel/builder/runtime.Dockerfile)：默认 EROFS runtime image构建。
- [`deploy/terraform/aliyun/values-akernel.yaml.tmpl`](../../akernel/deploy/terraform/aliyun/values-akernel.yaml.tmpl)：Dragonfly proxy注入。
- [`deploy/terraform/aliyun/values-dragonfly.yaml.tmpl`](../../akernel/deploy/terraform/aliyun/values-dragonfly.yaml.tmpl)：seed-client与worker client开关。

### 14.2 AKernel 论文草稿

- [`sections/2-background.tex`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/2-background.tex)：FaaS/Agent差异、四类状态、镜像和C/R问题。
- [`sections/5-design.tex`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/5-design.tex)：Lazy Load、Dragonfly、APlane、Checkpoint/Restore/Fork设计及TODO。
- [`sections/6-implementation.tex`](../../AKernel-Programmable-Datacenter-Scale-Infrastructure-for-Agents/sections/6-implementation.tex)：planned object metadata与scheduler边界。

### 14.3 AgentENV / OverlayBD 对照源码

- [`src/image/oci_image.rs`](../../kvcache-ai/AgentENV/src/image/oci_image.rs)：标准 OCI conversion与OverlayBD-native remote path。
- [`src/image/resolver.rs`](../../kvcache-ai/AgentENV/src/image/resolver.rs)：native referrer与conversion开关。
- [`storage/ublk/src/impls/overlaybd_target.rs`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs)：ublk block target。
- [`storage/overlaybd/src/backend/registryfs_v2.rs`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs)：registry Range/auth/retry/P2P facade。

### 14.4 官方源码与在线资料

- Nydus image-service：<https://github.com/dragonflyoss/image-service/tree/4345613858cc10e98149319130cb06adfb1f21e0>
- Nydus RAFS v5 layout：<https://github.com/dragonflyoss/image-service/blob/4345613858cc10e98149319130cb06adfb1f21e0/rafs/src/metadata/layout/v5.rs>
- Nydus RAFS v6/EROFS layout：<https://github.com/dragonflyoss/image-service/blob/4345613858cc10e98149319130cb06adfb1f21e0/rafs/src/metadata/layout/v6.rs>
- Nydus BlobCache：<https://github.com/dragonflyoss/image-service/blob/4345613858cc10e98149319130cb06adfb1f21e0/storage/src/cache/mod.rs>
- Nydus FUSE/fscache service：<https://github.com/dragonflyoss/image-service/tree/4345613858cc10e98149319130cb06adfb1f21e0/service/src>
- containerd nydus-snapshotter：<https://github.com/containerd/nydus-snapshotter/tree/8a7cfda01d877c33b2e4816689fe4e06cad865da>
- gVisor rootfs hints：<https://github.com/google/gvisor/blob/a29997e6c74152f06a69dda686f8196d9ba7d5b2/runsc/boot/mount_hints.go>
- gVisor EROFS implementation：<https://github.com/google/gvisor/tree/a29997e6c74152f06a69dda686f8196d9ba7d5b2/pkg/sentry/fsimpl/erofs>
- gVisor overlay configuration：<https://github.com/google/gvisor/blob/a29997e6c74152f06a69dda686f8196d9ba7d5b2/runsc/boot/vfs.go>
- Dragonfly项目：<https://github.com/dragonflyoss/dragonfly>
- Dragonfly架构：<https://d7y.io/docs/next/operations/architecture/architecture/>

### 14.5 论文

1. Z. Wang et al., “FaaSNet: Scalable and Fast Provisioning of Custom Serverless Container Runtimes at Alibaba Cloud Function Compute,” USENIX ATC 2021. 本地：[PDF](<../References/Wang et al_2021_{FaaSNet}.pdf>)，在线：<https://www.usenix.org/conference/atc21/presentation/wang-ao>。
2. H. Li et al., “DADI: Block-Level Image Service for Agile and Elastic Application Deployment,” USENIX ATC 2020. 本地：[PDF](<../References/Li 等 - 2020 - DADI Block-Level Image Service for Agile and Elastic Application Deployment.pdf>)，在线：<https://www.usenix.org/conference/atc20/presentation/li-huiba>。
3. Q. Chen et al., “YuanRong: A Production General-Purpose Serverless System for Distributed Applications in the Cloud,” ACM SIGCOMM 2024. 本地：[PDF](<../References/Chen 等 - 2024 - YuanRong A Production General-purpose Serverless System for Distributed Applications in the Cloud.pdf>)。
4. AKernel已有镜像对比：[AKernel Nydus、AgentENV OverlayBD 与 Agent Substrate Survey](./20260805T073718Z-akernel-nydus-agentenv-overlaybd-substrate-rl-rollout-image-lazy-load-survey.md)。
5. OverlayFS/DADI/OverlayBD基础：[DADI block layer lazy pulling解释](./20260806T070035Z-dadi-overlayfs-overlaybd-block-layer-lazy-pulling-explained-survey.md)。
