# 从 OverlayFS 到 OverlayBD：DADI 的块级 Overlay 与 Lazy Pulling 逐层解析

> 调研时间：2026-08-06 15:00:35（Asia/Shanghai）
>
> 文档时间戳：`20260806T070035Z`
>
> DADI论文：Huiba Li et al., *DADI: Block-Level Image Service for Agile and Elastic Application Deployment*, USENIX ATC 2020
>
> OverlayBD backstore commit：`a4807d1822cdb5df33204318ab325b0848f13c10`
>
> Accelerated Container Image / overlaybd-snapshotter commit：`4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9`
>
> AgentENV commit：`6296bc4be7ad79eb3a278eb5264ef011c341adf5`
>
> 方法：交叉核对DADI论文、OCI Image/Runtime规范、Linux OverlayFS与Docker官方文档、containerd Snapshotter接口、上游OverlayBD源码以及AgentENV Rust实现。DADI论文PDF共15个物理页；物理第1页为USENIX封面，物理第2页对应印刷p.727，故正文页码换算为“印刷页=PDF物理页+725”。

## 1. 先建立一个不会混淆的心智模型

### 1.1 最重要的七个结论

1. **Docker镜像层、OverlayFS层和OverlayBD层都叫“layer”，但寻址对象不同。** OCI/Docker layer描述路径级文件变化；OverlayFS按文件名/inode把多个目录合成一个目录；OverlayBD按逻辑块地址LBA把多个block changeset合成一个虚拟块设备。

2. **`runc`不负责pull镜像，也不实现OverlayFS。** Docker/containerd的image service负责获取manifest和blob，snapshotter/storage driver负责把layer变成可挂载rootfs；`runc`接收已经准备好的OCI bundle和rootfs目录，再创建namespace、mount并启动进程。

3. **传统`overlay2`的“overlay”是文件级newest-wins。** 相同路径存在于upper和lower时看upper；只在lower中的文件可以直接读；第一次写lower文件通常触发`copy_up`，把整个文件复制到upper后再改。删除用whiteout隐藏lower对象，而不是修改只读lower。

4. **DADI的核心不是“远程OverlayFS”，而是把newest-wins下沉到block层。** ext4仍然负责`/usr/bin/python`、inode、目录、权限和hardlink；OverlayBD只负责回答“这个LBA的最新数据在哪个layer blob的哪个offset”。因此它不需要实现POSIX文件系统语义。

5. **Lazy pulling不是完全不pull，而是先pull元数据，再在第一次真实I/O时pull数据range。** 标准OCI tar/gzip是顺序归档，不能高效随机读取任意文件对应的压缩字节；DADI必须先转换成带block index、可seek压缩和校验信息的OverlayBD-native/ZFile格式。

6. **DADI block upper避免的是文件级whole-file copy-up，不代表一次4 KiB应用写只产生精确4 KiB流量。** 上层ext4还可能写inode、目录、位图和journal；优势是这些都以实际写到VBD的block为单位进入upper，而不是先复制整个大文件。

7. **今天的“OverlayBD”不只有一种接法。** DADI论文支持完整block-level writable upper；当前开源overlaybd-snapshotter默认仍可把只读OverlayBD设备作为OverlayFS lower、用host目录作为writable upper；AgentENV则用Rust OverlayBD upper和ublk直接给Firecracker提供块设备。必须区分论文设计、上游默认配置和具体产品实现。

### 1.2 一张总图

~~~mermaid
flowchart TB
    subgraph FILE[传统 Docker/containerd + overlayfs]
      OCI[OCI tar/gzip layers\npath changesets]
      UNPACK[pull完整blob\nverify + decompress + apply]
      DIR[多个unpacked layer directories]
      OFS[OverlayFS\nlowerdirs + upperdir + workdir]
      ROOT1[merged rootfs directory]
      OCI --> UNPACK --> DIR --> OFS --> ROOT1
    end

    subgraph BLOCK[DADI / OverlayBD]
      OBD[OverlayBD-native layers\nLBA range changesets]
      META[先取header/index/ZFile metadata]
      VBD[OverlayBD virtual block device\nmerged LBA view]
      FS[ext4/XFS等普通文件系统]
      ROOT2[rootfs directory]
      OBD --> META --> VBD --> FS --> ROOT2
      OBD -. first block read .-> VBD
    end

    ROOT1 --> RUNC[runc或其他runtime]
    ROOT2 --> RUNC
~~~

### 1.3 五个词先分清

| 词 | 本文中的准确含义 | 不是 |
|---|---|---|
| OCI image layer | tar形式的文件系统changeset，描述add/modify/remove | 已挂载文件系统、block snapshot |
| OverlayFS lower/upper | Linux内核看到的目录层 | registry中的压缩blob |
| containerd snapshot | snapshotter管理的一份文件系统状态，具有parent/active/committed生命周期 | 一定包含进程内存的VM snapshot |
| OverlayBD layer | LBA range到layer blob offset的映射和对应数据 | 一组可以直接`ls`的文件 |
| lazy pull | 将data blob的传输延迟到实际read，并缓存/prefetch | 零网络、零本地缓存、任意OCI原地随机读 |

## 2. 从 OCI 镜像层开始：registry里存的到底是什么

### 2.1 一层不是“一个完整rootfs”

OCI Image规范把layer定义成一个文件系统changeset。它通常是tar，可能再用gzip或zstd压缩；changeset只包含相对父层的：

- additions；
- modifications；
- removals，使用`.wh.<name>`和`.wh..wh..opq`等whiteout条目表达。

因此下面三层：

~~~text
L1: ADD /usr/bin/python, /etc/base.conf
L2: MODIFY /etc/base.conf, ADD /app/main.py
L3: DELETE /etc/base.conf, ADD /app/config.yaml
~~~

不是三个完整目录的网络副本，而是三个可依次应用的文件变化包。OCI规范要求layer changeset被“apply”，不能把whiteout简单当普通空文件解压。[OCI Image Layer规范](https://github.com/opencontainers/image-spec/blob/e7f7c0ca69b21688c3cea7c87a04e4503e6099e2/layer.md#change-types)

### 2.2 content blob和运行时rootfs是两份不同形态

传统containerd大致维护两类数据：

1. **Content store**：registry下载的compressed、content-addressed blobs；用于digest校验、分发和GC。
2. **Snapshotter store**：把layer解压/apply后的可挂载文件系统状态；OverlayFS snapshotter通常表现为多个目录。

这解释了一个常见疑问：明明registry中已经有`layer.tar.gz`，为什么还要占用额外磁盘和CPU去unpack？因为Linux不能把普通tar.gz直接当一个支持任意pathname lookup、mmap、page fault和random read的rootfs使用。containerd的Snapshotter接口示例也明确写成`Prepare → mount → unpackLayer/apply → Commit`。[containerd `snapshotter.go`](https://github.com/containerd/containerd/blob/207ad711eabd375a01713109a8a197d197ff6542/core/snapshots/snapshotter.go#L180-L280)

### 2.3 标准冷启动为什么形成waterfall

~~~mermaid
sequenceDiagram
    participant U as docker/nerdctl/CRI
    participant I as image service
    participant R as Registry
    participant S as snapshotter
    participant C as runtime/runc

    U->>I: pull image ref
    I->>R: GET manifest/config
    loop 每个缺失layer
      I->>R: 下载完整compressed blob
      I->>I: digest验证
      I->>S: Prepare active snapshot
      I->>S: decompress + apply tar/whiteout
      I->>S: Commit snapshot
    end
    U->>S: Prepare container writable snapshot
    S-->>U: rootfs mounts/directory
    U->>C: OCI bundle + prepared rootfs
    C->>C: namespace/mount/pivot_root/start
~~~

DADI论文把这个串行依赖概括为：

~~~text
download whole image → unpack image → start container
~~~

下载消耗网络；解压/apply同时消耗CPU、page cache、host filesystem metadata IOPS和磁盘写带宽。只有这些完成，传统snapshotter才有可交给runtime的rootfs。论文的动机是把它改成：

~~~text
fetch small metadata → expose mountable image → fetch touched data on demand
~~~

### 2.4 `runc`在这条链上处于哪里

`runc`官方定位是“按照OCI规范spawn和run Linux container的CLI”。它要求调用者先准备OCI bundle，其中`config.json.root.path`指向已经存在的rootfs目录；README示例甚至先用`docker export | tar`填充`rootfs`，然后才调用runc。[runc README](https://github.com/opencontainers/runc/blob/0b9fa21be2bcba45f6d9d748b4bcf70cfbffbc19/README.md#L167-L205) [OCI Runtime bundle](https://github.com/opencontainers/runtime-spec/blob/36852b0d072a4b5da675300a9e73bc4b0853f5c6/bundle.md#L1-L20)

所以更准确的职责链是：

| 组件 | 主要职责 | 是否理解image layer格式 |
|---|---|---|
| Registry | 保存manifest/config/content blobs，支持HTTP传输 | 是，理解descriptor/media type，不负责mount |
| Docker/containerd image service | 解析image、fetch、校验、触发unpack | 是 |
| storage driver/snapshotter | 提供active/committed snapshot和rootfs mount | 是，或通过annotations识别remote format |
| OverlayFS/OverlayBD | 提供文件级或块级merged data path | 不负责容器进程生命周期 |
| runc | 使用已准备rootfs启动容器进程 | 通常不负责registry pull/unpack |

把“Docker使用overlay2”说成“runc使用overlay2”在口语上能猜到意思，但架构上不准确。

## 3. OverlayFS/overlay2：文件级overlay到底在做什么

### 3.1 lowerdir、upperdir、workdir和merged

Linux OverlayFS把一个或多个lower目录与一个upper目录合并成单一merged目录：

~~~text
mount -t overlay overlay \
  -o lowerdir=/l3:/l2:/l1,upperdir=/upper,workdir=/work \
  /merged
~~~

- `lowerdir`：只读视角的父层；多个lower从左到右优先级递减。
- `upperdir`：本容器的可写层。
- `workdir`：OverlayFS执行copy-up、rename等内部原子操作的工作目录，必须与upper在同一文件系统。
- `merged`：容器最终看到的rootfs目录。

Linux官方文档给出的基本mount结构见[OverlayFS文档](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst#L83-L138)。Docker `overlay2`把image layers作为lowerdirs，为每个container建立独立upper；当前官方文档说明它原生支持最多128个lower OverlayFS layers。[Docker OverlayFS driver](https://github.com/docker/docs/blob/f1dc8810dca8125bd334c76f8c831654f66791e6/content/manuals/engine/storage/drivers/overlayfs-driver.md#how-the-overlay2-driver-works)

~~~mermaid
flowchart TB
    M[/merged: 容器看到的rootfs/]
    U[/upperdir: container private writes/]
    L3[/lower L3: newest image layer/]
    L2[/lower L2/]
    L1[/lower L1: base/]
    W[/workdir: overlayfs internal/]

    M --> U
    M --> L3
    M --> L2
    M --> L1
    U --- W
~~~

### 3.2 read lookup：按pathname选择最高层对象

假设：

~~~text
L1: /etc/app.conf = "base"
L2: /app/main.py
L3: /etc/app.conf = "image-v3"
upper: initially empty
~~~

容器读`/etc/app.conf`时，OverlayFS在upper和lowers中查找同名dentry；upper没有，L3有，于是读L3的文件。之后若upper也有`/etc/app.conf`，upper版本遮蔽全部lower版本。

目录是特殊情况：upper/lower同名对象都是目录时，名称列表可以merge；普通文件、symlink和device等非目录对象直接由最高优先级对象胜出。这里的key是**路径/目录项/文件对象**，不是磁盘block地址。

### 3.3 write path：第一次写为什么会copy整个文件

lower被多个镜像或容器共享，不能原地修改。第一次以写方式打开一个只存在于lower的普通文件时，OverlayFS通常执行：

1. 确保upper中的父目录存在；
2. 在upper创建同类型对象并复制owner/mode/mtime/xattr；
3. 对普通文件复制完整data；
4. 后续读写都直接进入upper副本。

Linux官方文档还指出，metadata change和创建hardlink也可能触发copy-up。[Linux OverlayFS non-directories](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst#L263-L289)

考虑一个1 GiB的`/models/index.dat`，容器只修改offset 512 MiB处的4 KiB：

~~~text
OverlayFS default file-level copy-up:
  lower 1 GiB file --copy entire file--> upper 1 GiB file
  then overwrite 4 KiB in upper

DADI block upper:
  filesystem emits actual block writes
  append changed data/metadata blocks + LBA mappings to upper
~~~

这正是DADI强调“大文件小修改”和`chmod`等metadata operation的原因。需要加三个限定：

- 现代OverlayFS有`metacopy`等可选优化，纯metadata change不一定永远复制file data；DADI论文比较的是其2020生产配置，不能把whole-file copy-up写成所有内核/配置下不可改变的定律。
- Linux copy-up会先尝试`vfs_clone_file_range`；底层同一reflink-capable filesystem可能共享physical extents。逻辑上仍会建立独立upper inode/完整文件视图，并不是OverlayBD式按LBA维护的通用layer index。[内核`copy_up.c`](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/fs/overlayfs/copy_up.c#L261-L365)
- DADI下ext4可能产生journal和metadata write amplification，所以“应用改4 KiB”不等于“layer恰好只增长4 KiB”。

### 3.4 删除为什么需要whiteout

lower只读，`rm /etc/app.conf`不能真的删除L3中的文件。OverlayFS在upper创建whiteout，lookup看到它后隐藏同名lower对象；删除lower目录时会使用opaque directory等标记。Linux内核内部whiteout可表示为0/0字符设备或带`trusted.overlay.whiteout` xattr的零长度文件。[Linux OverlayFS whiteout](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst#L140-L157)

注意OCI tar中的`.wh.foo`是**传输格式的删除标记**，OverlayFS运行时whiteout是**内核merged filesystem的删除表示**。snapshotter在apply OCI layer时需要把前者转换成后者或等价文件系统状态。

### 3.5 OverlayFS的重要优点：共享lower page cache

多个容器可以mount同一个lower path；读同一lower文件时通常共享底层inode的host page cache。Docker官方文档把这列为overlay2适合高密度场景的重要原因。[Docker page caching](https://github.com/docker/docs/blob/f1dc8810dca8125bd334c76f8c831654f66791e6/content/manuals/engine/storage/drivers/overlayfs-driver.md#page-caching)

这也是DADI论文承认的一项反向tradeoff：原始DADI给每个容器创建独立VBD/文件系统，同一shared layer中的file可能在host看来属于不同device/page，未必像OverlayFS那样自然共享file page cache。论文把shared block pool + DAX作为future work，而不是已实现能力。

## 4. 为什么标准tar/gzip很难直接做lazy pull

### 4.1 lazy rootfs需要random access

应用第一次读取`/usr/lib/python3.11/json/__init__.py`时，lazy image service必须快速回答：

~~~text
这个文件的metadata和data位于哪个remote blob？
blob的哪些byte ranges？
这些ranges如何独立校验和解压？
~~~

标准tar主要是顺序entry stream；gzip又让后续未压缩位置依赖前面的压缩流状态。即使知道目标文件名，也通常不能只发一个小HTTP Range就独立解出它。DADI论文因此明确说remote image需要random read，而standard layer tarball需要改变格式（PDF物理p.3/印刷p.728）。上游Accelerated Container Image README也直接说明普通tarball不支持seek，必须完整下载；OverlayBD只加载各层index，再按映射远程fetch。[上游README](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/README.md#L113-L127)

### 4.2 “lazy image”常见的三条设计路线

| 路线 | 索引的key | 文件语义由谁处理 | 代表 |
|---|---|---|---|
| eager unpack + union FS | pathname；启动前已完整本地化 | host OverlayFS | Docker overlay2 |
| file/chunk-level lazy | inode/path → blob chunk/range | remote/lazy filesystem | CRFS/eStargz、Nydus/RAFS |
| block-level lazy | LBA range → layer blob range | VBD上的ext4/XFS等 | DADI/OverlayBD |

文件级lazy更直接知道“打开了哪个文件”，适合按file chunk做去重和prefetch；block级lazy对上层FS无感，天然适合virtio-blk、QEMU/Firecracker和不同guest filesystem，但看到的是metadata/data block访问而不是pathname语义。

### 4.3 lazy只是移动关键路径

传统路径把成本放在start之前；lazy路径把未触达数据省掉，但把miss latency放进第一次file open/page fault：

~~~text
Eager:
  T_ready = full_download + full_unpack + mount + runtime_start
  T_first_read ≈ local I/O

Lazy:
  T_ready = metadata/index + attach + mount + runtime_start
  T_first_read = index_lookup + remote_range + verify + decompress + cache
~~~

因此评价lazy pulling必须同时测“容器ready”和“first useful request/action”。若workload最终顺序扫描整个镜像，总下载量仍趋近完整image，还会叠加Range request、解压和cache管理成本。

## 5. DADI如何把文件层转换成block层

### 5.1 最关键的抽象翻转

DADI把完整image看成一块固定virtual size的磁盘，在上面创建ext4等普通文件系统。每个container image layer仍然语义对应一次Dockerfile/file changeset，但物理表示改为“apply这次文件变化时，底层VBD有哪些LBA ranges被改写”。

~~~mermaid
flowchart LR
    T[OCI layer tar\nadd/modify/whiteout]
    V[挂载父层形成的VBD + ext4]
    A[apply tar到ext4\n解释whiteout]
    W[捕获ext4向VBD发出的writes]
    B[immutable OverlayBD layer\nheader + changed blocks + index + trailer]

    T --> A
    V --> A
    A --> W --> B
~~~

论文的`.tgz → DADI`转换从最低层到最高层执行：为每层创建writable upper，解包tar并处理whiteout，然后commit成新的只读block layer。换句话说，普通OCI并不能凭空获得block-level random access；**转换成本被移到发布/首次转换阶段**。

### 5.2 文件系统没有消失，只是边界变了

~~~text
应用看到： /app/main.py, stat(), hardlink, xattr
              ↓
ext4看到： inode table、directory block、data block、journal
              ↓
OverlayBD看到： read/write LBA 12345..12352
~~~

所以论文说“filesystem agnostic”的准确含义是：OverlayBD核心不解析POSIX pathname/inode语义，可以承载ext4、XFS或guest支持的其他FS。它不意味着：

- image中没有文件系统；
- 随便换FS仍能读同一已构建image；
- guest不需要对应FS driver；
- malicious filesystem metadata不再需要验证。

### 5.3 block deletion不再需要运行时文件whiteout

当容器在ext4中删除一个文件，ext4更新directory entry、inode bitmap等blocks；这些新block版本被写入top OverlayBD layer。之后ext4看到的磁盘状态里文件已经不存在。OverlayBD不需要理解“删除文件”这个概念。

但在**从OCI tar转换**时，converter仍必须识别`.wh.foo`，在挂载的ext4视图中执行相应删除，才能产生正确block changeset。DADI论文§4.3明确描述了这个转换步骤（PDF物理p.10/印刷p.735）。

### 5.4 DADI layer也不是普通block snapshot

传统volume snapshot往往属于某块volume的point-in-time lineage，删除volume可能连带snapshot；container image layer则是可由许多image/租户引用、可通过registry分发的独立artifact。DADI保留后一种layer sharing模型，只把artifact内容编码为block delta。

它也不同于简单串很多qcow2 backing files：论文认为逐层查询会使read随depth退化，DADI用预合并index消除每次read的线性layer walk。

## 6. OverlayBD的layer blob和Segment index

### 6.1 immutable layer格式

DADI论文中的layer blob由：

~~~text
4 KiB header
raw data blocks written by this layer
sorted segment index
4 KiB trailer
~~~

组成。当前上游格式规范仍采用相同骨架；一个on-disk mapping为128 bit，记录：

- logical `offset`：image LBA起点；
- `length`：连续sector数量；
- `moffset`：这些数据在当前blob中的物理起点；
- `zeroed`：该range读零，不需要data mapping；
- `tag`：merged runtime中标记来源layer。

见[上游LSMT格式规范](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/format_spec.md#overview)。论文Figure 3使用当时的48/16/48/12/4-bit布局；当前上游和AgentENV Rust代码已经演进为50/14/55/1/8-bit布局。因此应保留“16-byte variable range mapping”的核心概念，不应把2020 bit allocation当成永远不变的ABI。[AgentENV `DiskSegmentMapping`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/format.rs#L7)

### 6.2 一个具体的三层例子

假设virtual disk的LBA 0–99：

| Layer | mappings | 含义 |
|---|---|---|
| L0 base | `0–99 → blob0@1000` | base提供全部磁盘内容 |
| L1 | `20–29 → blob1@200` | 中间层改写10个sector |
| L2 newest | `25–27 → blob2@80`; `50 → ZERO` | 最新层再次覆盖一部分，并discard LBA50 |

最终merged view不是把数据复制成一个100-sector新文件，而只是形成类似映射：

| logical range | source |
|---|---|
| 0–19 | L0 |
| 20–24 | L1 |
| 25–27 | L2 |
| 28–29 | L1 |
| 30–49 | L0 |
| 50 | ZERO |
| 51–99 | L0 |

读LBA 24–31时，index lookup返回三个segments，OverlayBD分别从L1、L2、L1、L0对应blob offsets读取并拼到caller buffer。没有pathname lookup，也没有为了merge而复制全部underlying data。

`moffset`会随segment裁剪同步前移。例如L1的`20–29`若映射到`blob1`的sector 200，最终只读取逻辑`28–29`时，应从物理`moffset=208`开始，而不是重新从200读。这就是mapping同时保存logical offset和blob offset的原因。

### 6.3 为什么要预合并index

若每次read从newest layer向下逐层查，`n`层、每层`m`条mapping的成本约为`O(n log m)`。DADI在image launch/load index阶段从top向下填充hole，得到带source layer tag的merged index；此后一次read只做merged range lookup，再按tag读取相应blob。

论文对1,664个layers、205个生产应用统计：merged index都不超过4.5K segments，约72 KiB；segment数与layer depth没有明显相关。4.5K index时单核超过6M query/s，而系统I/O峰值远低于此，所以当时index CPU不是主要瓶颈（Figures 6–7，PDF物理p.6–7/印刷p.731–732）。

当前AgentENV实现也把index lookup结果中的hole填零，把非zero mapping转换成`layer[tag].read(physical_offset)`。[`readonly.rs:199-304`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/file/readonly.rs#L199)

## 7. Lazy pulling的完整读路径

### 7.1 “pull完成”时本地究竟有什么

DADI 2020原型为了兼容当时的container engine，用一个很小的DADI metadata tarball“伪装”image pull完成；WordPress实验中标准`.tgz`为165 MiB、解包501 MiB、DADI LZ4格式274 MiB，而这个metadata tar只有9 KiB。它不是把274 MiB变成9 KiB，而是把其余数据留在registry/shared storage/P2P，等待read。

当前开源集成不再只靠论文中的fake tar描述：

- OCI descriptor annotations保存OverlayBD blob digest/size/fs type；
- `rpull`默认不下载layer data，给snapshotter传image/snapshot labels；
- snapshotter创建remote block snapshot并attach OverlayBD设备。

上游示例明确写着“`rpull` doesn't pull layer data”。[on-demand pulling示例](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/docs/EXAMPLES.md#L45-L99) 标签定义见[`pkg/label/label.go`](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/pkg/label/label.go#L19-L99)。

### 7.2 一次`open/read`如何变成HTTP Range

~~~mermaid
sequenceDiagram
    participant A as App
    participant F as ext4/guest filesystem
    participant B as VBD: vrbd/TCMU/ublk
    participant O as OverlayBD merged index
    participant Z as ZFile
    participant C as local block/blob cache
    participant R as Registry/P2P/shared storage

    A->>F: read("/usr/lib/libX.so", offset, len)
    F->>F: pathname/inode/page-cache lookup
    F->>B: READ logical sectors
    B->>O: read_at(LBA range)
    O->>O: merged index lookup
    loop 每个mapping
      O->>Z: pread(layer[tag], moffset, length)
      Z->>Z: jump-table定位compressed chunks
      Z->>C: read encoded byte range
      alt cache hit
        C-->>Z: cached bytes
      else cache miss
        C->>R: HTTP Range/P2P block request
        R-->>C: encoded bytes
      end
      Z->>Z: CRC verify + decompress touched chunks
      Z-->>O: raw block bytes
    end
    O-->>F: VBD read completion
    F-->>A: file bytes
~~~

论文cgroups原型的数据面是`app → ext4 → vrbd → user-space lsmd → local/P2P`；QEMU路径把filesystem/VBD放在guest，QEMU block driver访问host blob/P2P。论文只实现并评价了cgroups和QEMU，对Firecracker、gVisor和OSv只称技术上可接入，不能写成已经实现（Figure 12–13，PDF物理p.9/印刷p.734）。

### 7.3 ZFile为什么是lazy pulling的关键一层

LSMT/OverlayBD index解决“logical disk range来自哪个layer blob offset”，但若整个blob再用单一gzip流压缩，还是不能独立解压任意range。ZFile进一步：

1. 把原始blob分成固定大小的logical chunks；
2. 每个chunk独立压缩；
3. 保存jump table/index，将chunk number映射到encoded offset/length；
4. 可为每chunk保存CRC；
5. read只fetch并解压覆盖请求的chunks。

代价是：

- 首尾partial read会产生chunk read amplification；
- 独立chunk压缩通常弱于全流gzip，所以ZFile可能比`.tgz`大；
- high queue depth下decompression可能使CPU饱和；
- index/header/trailer本身仍需先取回。

AgentENV当前Rust ZFile会把相邻encoded blocks合成一个range batch，减少HTTP请求；jump table决定`batch_begin/batch_end`，随后逐块CRC/解压。[`zfile.rs:962-1060`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L962)

### 7.4 当前AgentENV如何把mapping落实成HTTP

AgentENV的ublk target把kernel block request换算成byte offset/length，read直接调用`ImageFile.read_at_into_with_ctx`，write进入writable image。[`overlaybd_target.rs:161-223`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L161)

remote `RegistryFileImplV2`最终构造：

~~~text
Range: bytes=<offset>-<offset+count-1>
~~~

并处理redirect/auth/retry，把response chunks填入caller buffer。[`registryfs_v2.rs:704-876`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L704) [`registryfs_v2.rs:1542-1600`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L1542)

这段代码非常直观地证明了lazy pull的本质：不是container create时循环下载所有blob，而是VBD read已经确定`blob offset + length`后才发对应HTTP Range。

## 8. Writable block upper：读、写、discard和commit

### 8.1 append-only upper

DADI的log-structured writable layer由一个data file和一个index file组成：

~~~text
write LBA X..Y:
  append raw bytes to data EOF
  append/update mapping X..Y -> new physical EOF
  in-memory mutable index让new mapping立即生效

read LBA X..Y:
  first query mutable upper
  holes fall through to readonly merged lower index
~~~

因为raw data和mapping都append，随机overwrite被转换为顺序写；旧mapping指向的数据成为garbage。上游C++实现的`pwrite`确实先append data，再插入`SegmentMapping`和append index。[`file.cpp:789-870`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/file.cpp#L789-L870)

### 8.2 TRIM/discard

TRIM不是去修改所有lower layers，而是在upper插入`zeroed` mapping。之后该LBA range由top layer明确返回零，遮蔽lower旧数据。这与OverlayFS whiteout具有相似的“tombstone”思想，但key不同：前者遮蔽LBA range，后者遮蔽pathname。

### 8.3 commit与GC

overwrite会留下不再被latest mapping引用的garbage。DADI可以：

- 后台把live data复制到新文件并回收旧文件；
- commit时按LBA排序、合并相邻continuous mappings，把live view变成immutable readonly layer；
- 在image build时使用`stack-and-commit`，先叠新upper继续构建，后台commit旧upper。

因此block upper不是“没有copy”，而是把copy从每次first file write的whole-file copy-up，转换为按实际changed blocks append，以及可调度的GC/commit compact。写密集且反复overwrite时，garbage、GC、commit I/O和空间峰值都必须测量。

### 8.4 再看1 GiB文件例子

| 操作 | 默认file-level OverlayFS | DADI block upper |
|---|---|---|
| 首次改1 GiB文件中的4 KiB | 先whole-file copy-up，再写4 KiB | 记录ext4实际发出的changed blocks/mappings |
| `chmod` lower file | 通常触发copy-up；现代metacopy可优化 | ext4修改相关metadata blocks |
| 删除lower file | upper whiteout/opaque marker | ext4 metadata变化成为new LBA versions |
| 重复覆盖同一位置 | upper文件原地write | append新data/mapping，旧data成为garbage |
| 提交成新image layer | 计算/打包file changeset tar | compact live mappings/data为sealed block layer |

这说明两者不是“一个永远快、一个永远慢”：OverlayFS成熟、page-cache共享自然、重复写upper没有append garbage；OverlayBD对大文件小修改、remote random access、VM block接口和分层镜像统一更有吸引力。

## 9. 一个容易忽略的现实：OverlayBD与OverlayFS可以同时出现

### 9.1 当前上游默认路径是“block lower + file upper”

DADI论文§3.4把log-structured block upper作为消除union filesystem依赖的完整方案。但当前开源`overlaybd-snapshotter`支持三种rootfs组合：

| mode | readonly image | container writable layer | 返回给runtime |
|---|---|---|---|
| `overlayfs`，当前默认 | OverlayBD VBD上挂载的FS目录 | host普通directory upper | `type=overlay, lowerdir=<obd mount>, upperdir=<fs>, workdir=<work>` |
| `dir` | OverlayBD lowers | OverlayBD writable upper | 已挂载block image的bind mount |
| `dev` | OverlayBD lowers | OverlayBD writable upper | block device + FS mount描述，适合secure container/VM |

默认值`RwMode: "overlayfs"`和storage type定义见[`pkg/snapshot/overlay.go:50-129`](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/pkg/snapshot/overlay.go#L50-L129)。实际mount构造见[`overlay.go:1171-1256`](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/pkg/snapshot/overlay.go#L1171-L1256)。

因此在一台现代containerd host上看到：

~~~text
overlay lowerdir=/.../block/mountpoint,
        upperdir=/.../fs,
        workdir=/.../work
~~~

并不表示OverlayBD没有工作。它已经把多个remote image layers在block层合成一个只读base rootfs；OverlayFS只负责本container的file-level writable upper。

### 9.2 为什么会选择这种混合路径

优点：

- remote readonly image仍可lazy pull；
- OverlayFS writable path成熟，容器runtime兼容性好；
- 多container可以共享同一个只读OverlayBD mount，而不必每个active container都建独占writable VBD；
- GC、quota和故障语义更接近传统containerd。

代价：

- 第一次写lower大文件仍可能触发OverlayFS whole-file copy-up；
- 不能获得DADI block upper的完整大文件小修改优势；
- data path同时出现block overlay与file overlay，排障时更复杂。

这进一步说明，`OverlayBD image format`、`OverlayBD readonly device`、`OverlayBD writable upper`是三个可以分别采用的能力。

## 10. Cache、Prefetch与P2P分别解决什么

### 10.1 Cache：避免同一node重复remote read

lazy pull没有cache时，每次读同一ZFile encoded range都要访问registry。合理实现会为immutable blob建立partial local cache：

~~~text
read encoded range
  -> query local presence bitmap/sparse extents
  -> hit: local disk/memory
  -> miss: one loader fetches remote range
  -> fill local sparse cache
  -> concurrent readers reuse result
~~~

上游OverlayBD file cache使用sparse file/fiemap或OCF管理partial blob；AgentENV的full-file cache为每个raw blob维护sparse data file、RoaringBitmap presence和per-block inflight dedup。同节点多个sandbox同步import相同依赖时，这种single-refill比单纯HTTP Range更关键。[AgentENV cache设计](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/cache/full_file_cache/cache_store.rs#L43)

需要区分三种“缓存”：

| 缓存 | 保存什么 | 命中后省什么 |
|---|---|---|
| raw remote blob cache | registry中的tar/ZFile encoded bytes | HTTP/P2P range和部分TLS/auth |
| premerged index cache | 多层index合并后的mapping artifact | attach时逐层load/merge index |
| filesystem/page cache | ext4看到的decompressed file/block pages | LSMT/ZFile和underlying I/O |

它们不是同一层，报告“cache hit”时必须说明是哪一种。

### 10.2 Prefetch：把可预测miss从串行变并行

DADI论文用一次已知启动的`blktrace`记录read trace，在另一个host用`fio`、iodepth 32并行replay read，同时启动新container。它将cold与warm startup差距减少95%（Figure 16，PDF物理p.11/印刷p.736）。

这不是“DADI自动知道未来读哪些文件”，而是offline trace replay proof-of-concept。生产化还要解决：

- image digest/runtime版本对应哪份trace；
- trace什么时候过期；
- 不同entrypoint/参数是否使用不同working set；
- prefetch与foreground miss如何抢带宽；
- 预取错误时的read amplification。

AgentENV当前也有trace Record/Replay wrapper，但上游的dynamic prioritized-file prefetch尚未完整迁移；不能仅凭存在`prefetch.rs`就推断具备自动file-level working-set learning。

### 10.3 P2P：解决all-cold nodes同时打爆registry

一个节点lazy read只减少单节点字节。如果1,000个cold hosts同时启动相同image，它们可能同步请求相同ranges，origin压力仍近似：

~~~text
number_of_cold_nodes × touched_working_set
~~~

DADI为每个layer blob维护一棵tree：

~~~mermaid
flowchart TB
    REG[Registry]
    ROOT[DADI-Root\nmetadata + root cache]
    A1[DADI-Agent A\nlocal partial cache]
    A2[DADI-Agent B]
    B1[DADI-Agent C]
    B2[DADI-Agent D]
    C1[DADI-Agent E]

    REG --> ROOT
    ROOT --> A1
    ROOT --> A2
    A1 --> B1
    A1 --> B2
    A2 --> C1
~~~

child miss沿parent向上查找，返回数据沿途缓存。它利用burst startup时hosts大致按相似顺序读取相同blocks的特征，做application-level multicast；与BitTorrent式rarest-first不同。parent失败时回Root重新选择，per-layer topology默认20分钟过期（Figure 11，PDF物理p.8–9/印刷p.733–734）。

P2P与lazy是正交的：

- lazy决定只取working set；
- cache决定同节点只取一次；
- P2P决定多节点不都从origin取；
- prefetch决定已知working set尽量并行提前取。

## 11. DADI论文的性能证据应该怎样读

### 11.1 实验环境与基线边界

论文物理服务器为dual-socket multi-core Xeon、10 GbE或更高；public-cloud VM为4 vCPU、8 GiB，vNIC burst 5 Gbps、sustained 1.5 Gbps。local storage一般为NVMe；“cloud disk”是限制到2,000 IOPS、100 MB/s的设备。每个test前清host/guest page cache和DADI persistent cache。

baseline包括`.tgz+overlay2`、CRFS、pseudo-Slacker、thin LVM和P2P whole-layer download。pseudo-Slacker实际是LVM+NFS近似，因为作者没有Tintri VMstore；当时CRFS还借用kernel overlayfs。这些不是完全同源、同成熟度实现。

### 11.2 关键数字

| 实验 | 论文结果 | 不能外推成什么 |
|---|---|---|
| WordPress image | 21 layers；`.tgz` 165 MiB；unpacked 501 MiB；DADI LZ4 274 MiB；metadata tar 9 KiB | 9 KiB不是完整image大小 |
| warm single instance | NVMe上DADI比OverlayFS/LVM快15%–25%；cloud disk上超过2× | 不是所有workload/现代kernel都固定如此 |
| trace prefetch | 消除95%的cold-vs-warm差距 | 不是在线自动预测 |
| batch 1→32 instances | pseudo-Slacker 1.5→2.3 s；DADI基本维持0.7 s | remote servers与host同datacenter |
| production pull | DADI metadata近半hosts≤0.2 s，其余约1 s；等价`.tgz`多数>20 s | “pull完成”没有包含后续所有lazy bytes |
| large scale | 1,000 VMs × 10 containers，10,000个在4 s内start | 不是单host 10,000，也不是只有runtime create |
| hyper-scale | tens of hosts构造root-to-leaf path后投影到100,000 | 100,000是projected，不是实测 |
| image build | 15 layers、7,944 files、545 MiB，DADI快20%–40% | 明确不含commit和compression |

Figure 14中的single cold-start柱状图没有正文精确原始数；若按图约读，`.tgz+overlay2≈11 s`、CRFS≈19 s、pseudo-Slacker≈3.8 s、DADI Registry≈2.1 s、DADI P2P Root≈0.7 s。引用时必须标“图中约读”，不应写成精确测量表。

### 11.3 I/O与compression

- 8 KiB uncached random read、QD=1时DADI与LVM相近；DADI随queue depth提升较慢，QD=128时uncompressed版本取得图中最高IOPS。
- ZFile在QD<32比uncompressed DADI快10%–20%，因为少传输的数据抵消decompression CPU；QD>32时CPU成为瓶颈。
- `du`与`tar`扫描中DADI优于overlay2/LVM，cloud disk收益更明显；正文没有给精确百分比。
- DADI uncompressed image包含ext4等FS overhead；对大于10 MiB layers通常相对`.tar`低于5%，但ZFile chunk compression通常仍比`.tgz`大。

这些结果说明compression有时能提升I/O吞吐，不是说“解压没有成本”。当storage/network慢、CPU富余时，少传bytes更重要；当高速local NVMe和高并发decompression耗尽CPU时，结论可能反转。

## 12. 论文、上游OverlayBD与AgentENV不是同一份代码

### 12.1 三代实现对照

| 维度 | DADI论文2020 | 当前上游OverlayBD | AgentENV当前源码 |
|---|---|---|---|
| container integration | Docker graph driver + containerd snapshotter | `accelerated-container-image` proxy snapshotter/converter | AgentENV自有ImageResolver/ublk daemon |
| block frontend | cgroups `vrbd`；QEMU block driver | TCMU + TCM loopback `/dev/sdX` | Linux ublk + io_uring `/dev/ublkN` |
| engine | user-space `lsmd` | C++/Photon OverlayBD | Rust async OverlayBD |
| layer mapping | 16-byte variable Segment，512 B unit | LSMT 50/14/55/1/8-bit mapping | 格式兼容的`DiskSegmentMapping` |
| remote read | shared storage/Registry/P2P | registryfs + cache | `registryfs_v2` HTTP Range + file cache/P2P |
| compressed random read | ZFile | ZFile | ZFile，精确batch/coalescing/retry metrics |
| writable | log-structured block upper | append/sparse及snapshotter多种mode | log/sparse/hybrid upper + seal/restack |
| snapshot use | 主要讨论image build/container upper | live restack能力 | rootfs/drive/memory snapshot统一使用 |

论文正文没有使用“LSMT”这个缩写；它称OverlayBD的user-space daemon为`lsmd`，把upper称为log-structured。今天源码中的LSMT是这套layered segment mapping/log-structured upper的实现名称，不应把它倒写成论文原术语，也不等同于数据库领域的传统LSM-tree。

### 12.2 上游仓库也分两层

- [`containerd/accelerated-container-image`](https://github.com/containerd/accelerated-container-image/tree/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9)：Go snapshotter、converter、ctr integration和labels。
- [`containerd/overlaybd`](https://github.com/containerd/overlaybd/tree/a4807d1822cdb5df33204318ab325b0848f13c10)：C++ native LSMT/ZFile/registryfs/cache/TCMU engine。

只看Go snapshotter看不到Segment lookup和ZFile解压；只看C++ engine也看不到containerd何时把remote annotations转换为snapshot mount。

### 12.3 AgentENV的两级地址转换

AgentENV remote read不是“LSMT直接发HTTP”，实际是：

~~~text
guest logical sector
  -> ublk request byte range
  -> LSMT merged mapping: choose layer[tag], uncompressed layer offset
  -> ZFile jump table: choose encoded blob byte range
  -> raw blob cache
  -> Registry HTTP Range / P2P
~~~

这两级mapping分别回答：

1. 最新guest disk block来自哪一个layer？
2. 这个uncompressed layer range在压缩blob里对应哪些encoded bytes？

AgentENV还把Firecracker pause产生的writable upper执行`close_seal + rename + fresh upper + restack`，快速把旧upper变成newest readonly lower；这不是DADI论文rootfs lazy pull本身，而是block-layer abstraction被复用于VM snapshot。

### 12.4 只有OverlayBD-native source才是AgentENV runtime lazy

AgentENV解析到OverlayBD-native manifest时只生成`repoBlobUrl + digest + size` remote descriptors，明确跳过blob download；运行时registryfs按需取。[`oci_image.rs:198-235`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L198)

普通OCI image若本地converted cache不全，则先用`regctl image copy`获取layers并逐层转换，完成后才得到local `.commit` lowers。[`oci_image.rs:236-275`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L236) [`oci_image.rs:338-438`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L338)

所以“OverlayBD支持lazy”不等于“给任意DockerHub tag就能立即lazy”。格式发布和conversion pipeline是系统的一部分。

## 13. 机制对比与代价总表

| 维度 | OverlayFS/overlay2 | DADI完整block upper | OverlayBD lower + OverlayFS upper |
|---|---|---|---|
| overlay key | pathname/file object | LBA range | lower按LBA，upper按pathname |
| startup前data | 标准路径完整pull+unpack | metadata/index即可attach | readonly base metadata/index即可 |
| file semantics | host OverlayFS处理 | VBD上的ext4/XFS处理 | lower FS + host OverlayFS |
| 大文件小修改 | first write通常whole-file copy-up | changed FS blocks进入upper | 仍可能file copy-up到host upper |
| shared lower page cache | 自然共享相同lower inode | 原论文per-VBD不自然共享 | 可共享同一mounted readonly base，需实测 |
| VM/secure runtime | 要传目录/virtio-fs/9p等 | block device天然匹配 | readonly block匹配，writable目录需额外路径 |
| random remote read | vanilla不支持 | ZFile + index + Range | readonly部分支持 |
| repeated overwrite | upper文件原位覆盖 | append garbage，需要GC/commit | upper目录原位覆盖 |
| format portability | 标准OCI最广 | 需要native转换/annotations | 同样需要native readonly image |
| failure surface | unpack与host FS metadata | remote first-touch、VBD daemon、cache | 两套data path组合 |

### 13.1 DADI真正解决的问题

- 避免把未触达image data放进启动关键路径；
- 避免每node对完整layer做decompress+untar+metadata creation；
- 让container和VM共享block image interface；
- 让大文件/metadata变更以block delta表达；
- 用merged index让多层read不随layer depth线性退化；
- 为block trace prefetch和P2P multicast提供统一range单位。

### 13.2 DADI没有自动解决的问题

- application/runtime自身初始化；
- all-cold nodes origin fan-out，除非cache/P2P有效；
- first-touch tail latency；
- ZFile CPU/read amplification；
- image conversion与版本管理；
- page-cache共享和double caching；
- writable upper garbage/commit；
- registry auth、Range正确性和数据完整性；
- workload最终读取全image时的总流量。

## 14. 常见误解问答

### Q1：DADI是不是把OverlayFS换成了另一个文件系统？

不是。OverlayBD是virtual block device/layer format；其上仍要mount ext4/XFS等文件系统。完整block upper模式可以不需要OverlayFS，但文件系统本身没有消失。

### Q2：`overlay2`是不是runc的一部分？

不是。传统Moby的overlay2是graphdriver；现代containerd有overlayfs snapshotter。它们准备rootfs mount，containerd shim挂到bundle/rootfs后，runc才启动容器。

### Q3：Docker image layer就是lowerdir吗？

语义上相关，但物理上不是同一东西。registry layer是tar/gzip changeset；unpack/apply以后，snapshotter目录才可以成为OverlayFS lowerdir。

### Q4：lazy pull以后`docker pull`为什么会很快？

因为“pull完成”的语义被remote snapshotter扩展为manifest、config、annotations/index等metadata ready，不表示所有layer data bytes已经本地化。真正bytes在后续read中到达。

### Q5：既然按block读取，OverlayBD怎么知道某文件在哪里？

它不知道。ext4先把pathname/inode操作翻译成block reads；OverlayBD只处理这些LBA reads。这既是简化，也是失去file semantic visibility的代价。

### Q6：改一个字节是否只下载/写一个字节？

不会。最小单位受到filesystem block、VBD sector、ZFile compression chunk、cache refill和read-ahead共同影响。论文mapping granularity为512 B，但常见ext4 I/O和ZFile chunk会更大。

### Q7：OverlayBD和qcow2有什么区别？

两者都能表示block mapping/CoW，但DADI layer面向container image artifact sharing和registry分发，并通过merged segment index优化很多layer的组合；不是简单为每个volume串一条qcow2 backing chain。

### Q8：P2P是不是lazy pull的必要条件？

不是。可以从registry或shared storage按需读；P2P主要解决大规模burst下origin/network fan-out。

### Q9：用了OverlayBD就一定比OverlayFS快吗？

不是。取决于conversion状态、工作集比例、network/cache、compression CPU、page-cache sharing、write pattern和runtime接口。DADI论文数字来自其2020软硬件与workload，不能直接当作当前AgentENV实测。

## 15. 两个可操作的学习实验

### 15.1 先亲手观察OverlayFS

在有root权限的测试机上：

~~~bash
mkdir -p /tmp/ovl/{lower,upper,work,merged}
dd if=/dev/zero of=/tmp/ovl/lower/bigfile bs=1M count=128
echo lower-only >/tmp/ovl/lower/message

mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged

cat /tmp/ovl/merged/message
printf X | dd of=/tmp/ovl/merged/bigfile bs=1 seek=4096 conv=notrunc
find /tmp/ovl/upper -maxdepth 2 -ls
du -h /tmp/ovl/lower/bigfile /tmp/ovl/upper/bigfile

rm /tmp/ovl/merged/message
find /tmp/ovl/upper -maxdepth 2 -ls

umount /tmp/ovl/merged
rm -rf /tmp/ovl
~~~

观察重点：

- read-only打开`message`时upper仍为空；
- 第一次改`bigfile`后upper出现独立文件；
- 删除`message`后lower原文件仍存在，upper出现whiteout表示；
- `work`不是应用cwd。

底层FS若支持reflink，physical allocation可能小于完整size，但logical upper inode/copy-up语义仍成立。

### 15.2 再观察OverlayBD lazy read

按上游Quickstart准备TCMU、overlaybd-snapshotter和一个OverlayBD-native image后：

~~~bash
# metadata-only remote pull
sudo /opt/overlaybd/snapshotter/ctr rpull <overlaybd-native-image>

# 确认此时没有完整layer data download
sudo ctr run --snapshotter=overlaybd --rm -t <image> demo

# 查看创建的block device与mount
lsblk
findmnt -t overlay,ext4

# 结合registry/cache metrics观察首次读取大文件前后bytes/range变化
~~~

实验报告至少区分：

- manifest/config/index bytes；
- remote encoded data bytes；
- decompressed guest block bytes；
- application requested bytes；
- cold cache与warm cache；
- create/ready latency与first useful request latency。

如果输入不是OverlayBD-native image，先进行完整OCI conversion会污染这个实验，必须单独计时。

## 16. 最终总结

理解DADI最短的一条逻辑链是：

~~~text
OCI/Docker layer：路径级文件changeset，适合registry分发
        ↓ traditional apply
OverlayFS：多个本地目录的文件级newest-wins视图
        ↓ DADI changes representation
OverlayBD：多个LBA range changeset的block级newest-wins视图
        ↓
ext4/XFS：把merged virtual disk重新解释成文件和目录
~~~

传统OverlayFS要求启动前先把tar changeset materialize成本地目录；DADI让OverlayBD index先描述整个virtual disk，把真实data blob留在远端。应用读取文件时，普通FS将请求翻译成LBA，merged segment index选择最新layer，ZFile再把uncompressed block range映射为可独立获取和解压的encoded range，cache miss才触发Registry/P2P读取。

所以DADI的创新不是取消layer，也不是取消filesystem，而是把layer的物理表示从“file diffs in directories”变成“block diffs behind a filesystem”，从而为fine-grained lazy transfer、seekable compression、block prefetch、P2P和VM/secure-container统一接口打开空间。代价则是native format conversion、remote first-touch、block/cache栈复杂度、writable GC以及原论文中不自然的跨VBD page-cache sharing。

## 17. 关键源码索引

### 17.1 传统容器路径

- OCI layer add/modify/remove/whiteout：[OCI Image Spec](https://github.com/opencontainers/image-spec/blob/e7f7c0ca69b21688c3cea7c87a04e4503e6099e2/layer.md#L1-L34) [whiteout](https://github.com/opencontainers/image-spec/blob/e7f7c0ca69b21688c3cea7c87a04e4503e6099e2/layer.md#L227-L287)
- Linux OverlayFS lower/upper/work/merged：[overlayfs.rst](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst#L83-L157)
- Linux whole-file logical copy-up：[overlayfs.rst](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst#L263-L289) [copy_up.c](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/fs/overlayfs/copy_up.c#L261-L365)
- Docker overlay2 mount与copy-up说明：[Docker Docs](https://github.com/docker/docs/blob/f1dc8810dca8125bd334c76f8c831654f66791e6/content/manuals/engine/storage/drivers/overlayfs-driver.md#how-container-reads-and-writes-work-with-overlay2)
- containerd Snapshotter lifecycle：[snapshotter.go](https://github.com/containerd/containerd/blob/207ad711eabd375a01713109a8a197d197ff6542/core/snapshots/snapshotter.go#L180-L332)
- containerd apply compressed layer：[apply.go](https://github.com/containerd/containerd/blob/207ad711eabd375a01713109a8a197d197ff6542/pkg/rootfs/apply.go#L81-L177)
- runc只消费已准备bundle/rootfs：[runc README](https://github.com/opencontainers/runc/blob/0b9fa21be2bcba45f6d9d748b4bcf70cfbffbc19/README.md#L167-L205)

### 17.2 上游OverlayBD

- 总体on-demand I/O路径：[Accelerated Container Image README](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/README.md#L113-L127)
- remote/local storage types和默认mode：[`overlay.go`](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/pkg/snapshot/overlay.go#L50-L129)
- OverlayBD-native annotations：[`label.go`](https://github.com/containerd/accelerated-container-image/blob/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9/pkg/label/label.go#L19-L99)
- 16-byte LSMT mapping：[`index.h`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/index.h#L32-L85)
- LSMT layer blob格式：[`format_spec.md`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/format_spec.md)
- merged mapping read：[`file.cpp:570-624`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/file.cpp#L570-L624)
- append-only writable upper：[`file.cpp:789-912`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/lsmt/file.cpp#L789-L912)
- ZFile格式：[`format_spec.md`](https://github.com/containerd/overlaybd/blob/a4807d1822cdb5df33204318ab325b0848f13c10/src/overlaybd/zfile/format_spec.md)

### 17.3 AgentENV Rust实现

- mapping格式：[`format.rs:7-54`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/format.rs#L7)
- top-wins ComboIndex：[`index.rs:595-668`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/index.rs#L595)
- mapping到layer read：[`readonly.rs:199-304`](../../kvcache-ai/AgentENV/storage/overlaybd/src/lsmt/file/readonly.rs#L199)
- ZFile range batching：[`zfile.rs:962-1060`](../../kvcache-ai/AgentENV/storage/overlaybd/src/compression/zfile.rs#L962)
- registry HTTP Range：[`registryfs_v2.rs:704-876`](../../kvcache-ai/AgentENV/storage/overlaybd/src/backend/registryfs_v2.rs#L704)
- ublk target：[`overlaybd_target.rs:161-223`](../../kvcache-ai/AgentENV/storage/ublk/src/impls/overlaybd_target.rs#L161)
- OverlayBD-native与standard OCI分支：[`oci_image.rs:198-275`](../../kvcache-ai/AgentENV/src/image/oci_image.rs#L198)

## 18. 参考资料

1. Huiba Li, Yifan Yuan, Rui Du, Kai Ma, Lanzheng Liu, Windsor Hsu. *DADI: Block-Level Image Service for Agile and Elastic Application Deployment*. USENIX ATC 2020. [USENIX](https://www.usenix.org/conference/atc20/presentation/li-huiba)；[本地PDF](<../References/Li 等 - 2020 - DADI Block-Level Image Service for Agile and Elastic Application Deployment.pdf>)。
2. Linux Kernel Documentation. *Overlay Filesystem*. [固定源码版本](https://github.com/torvalds/linux/blob/adc218676eef25575469234709c2d87185ca223a/Documentation/filesystems/overlayfs.rst)；[在线文档](https://docs.kernel.org/filesystems/overlayfs.html)。
3. Docker Documentation. *OverlayFS storage driver*. [固定源码版本](https://github.com/docker/docs/blob/f1dc8810dca8125bd334c76f8c831654f66791e6/content/manuals/engine/storage/drivers/overlayfs-driver.md)；[在线文档](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)。
4. OCI Image Format Specification. *Image Layer Filesystem Changeset*. [v1.1.0固定版本](https://github.com/opencontainers/image-spec/blob/e7f7c0ca69b21688c3cea7c87a04e4503e6099e2/layer.md)。
5. containerd. *Snapshotter API*. [v2.0.0固定版本](https://github.com/containerd/containerd/blob/207ad711eabd375a01713109a8a197d197ff6542/core/snapshots/snapshotter.go)。
6. Open Container Initiative. `runc`. [v1.2.0 README](https://github.com/opencontainers/runc/blob/0b9fa21be2bcba45f6d9d748b4bcf70cfbffbc19/README.md)。
7. containerd. *Accelerated Container Image / OverlayBD Snapshotter*. [GitHub固定版本](https://github.com/containerd/accelerated-container-image/tree/4f5c597f83acf8420cc54e1e9fcc2ff6b4e303f9)。
8. containerd. *OverlayBD native backstore*. [GitHub固定版本](https://github.com/containerd/overlaybd/tree/a4807d1822cdb5df33204318ab325b0848f13c10)。
9. Tyler Harter et al. *Slacker: Fast Distribution with Lazy Docker Containers*. FAST 2016. [USENIX](https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter)。
10. containerd. *Stargz Snapshotter*. [GitHub](https://github.com/containerd/stargz-snapshotter)。用于对照file-oriented seekable layer与remote snapshotter路线。
11. DragonflyOSS. *Nydus Image Service*. [GitHub](https://github.com/dragonflyoss/nydus)。用于对照RAFS file/chunk-level lazy image路线。
