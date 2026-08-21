# Firecracker microVM 内运行 Docker 并做 Snapshot/Restore：与 gVisor C/R 的对照验证

> 调研时间：2026-08-20（Asia/Shanghai）
> 文档时间戳：20260820T171500Z
> 前置文档：[gVisor DinD C/R 调研](20260820T082000Z-gvisor-dind-checkpoint-restore-survey.md)（同一台机器、同一套验证流程）
> 源码仓库：`/home/keyang/Hypervisors/firecracker`（上游 firecracker/firecracker，HEAD `6c5d9f3e0`，版本 `v1.17.0-dev`，本机自行编译 musl 静态版）
> 方法：本机（ubuntu-vm，Linux 5.15，47G 内存，/dev/kvm root:kvm 660）搭建 Firecracker microVM 实测；不动系统 dockerd 与任何远程服务器。

## 1. 结论摘要

**核心结论：microVM 的全量快照（CPU + 内存 + virtio 设备状态）完整保留了 guest kernel 的一切运行时状态——包括 guest 内运行时（dockerd）通过 netlink 创建的 `docker0` 网桥、veth 对和 network namespace。restore 之后不但旧容器原样续跑，还能继续新建 bridge 网络容器，数据面（容器间 ping）完全正常。这与 gVisor 进程级 C/R 形成鲜明对照。**

问题回顾（gVisor 侧结论）：*"如果 runtime 不是 gvisor 的 sentry，而是一个正儿八经的 microVM，guest kernel 维护的 docker0 和 netns 会被 restore 吗？"* —— **会，原样恢复，连 MAC 地址都不变。**

| 验证项 | gVisor C/R（前次） | Firecracker snapshot/restore（本次） |
|---|---|---|
| 快照粒度 | 进程级（sandbox 内所有 task 的寄存器/内存/文件/netstack 协议栈） | VM 级（全部 vCPU 寄存器 + 全部 guest 物理内存 + virtio 设备状态 + KVM 时钟） |
| 快照大小 / 耗时 | 59MB / ~2.4s（sandbox 常驻 ~200MB） | 4.0GB（memory.snap）+ 44KB（vmstate.snap）/ pause+create ~2s、load+resume ~10ms 级（API 日志：`'load snapshot' API request took 9881 us`） |
| 内层 dockerd 存活 | ✅ | ✅ |
| 内层业务容器状态（计数器） | ✅ tick 6→55 连续 | ✅ tick 66→70 连续（暂停窗口仅跳 2 个时间戳） |
| guest 内 `docker0`（bridge） | ❌ 丢失，只剩 lo + eth0 | ✅ **同 MAC（5e:32:9a:3c:c3:4f）同 IP（172.17.0.1/16）原样恢复** |
| 容器 veth（宿主侧 `vethffe5324`） | ❌ 丢失 | ✅ 原样恢复（`vethffe5324@if2`，MAC 不变） |
| 容器 netns（`/var/run/docker/netns/a731ac75c4ae`） | ❌ netns 内只剩 lo | ✅ 句柄与内容原样恢复 |
| **restore 后新建 bridge 网络容器** | ❌ 失败（gVisor 侧根因之一） | ✅ `post-restore-app` 正常创建，新 veth `veth74bd6fe` 挂上 docker0 |
| restore 后容器 ↔ 网关 / 容器 ↔ 容器连通 | —（无 bridge 可测） | ✅ ping 172.17.0.1 与 172.17.0.2 均 0% loss |
| restore 前置条件 | docker ≤ 27（moby#50750）、外层 daemon 私有栈 | **fresh Firecracker 进程**，restore 前不得配置任何资源（Logger/Metrics 除外）；设备配置全部由 vmstate.snap 携带 |
| 时钟行为 | — | guest 时钟从暂停点连续续走（uptime 连续，`random: crng reseeded due to virtual machine fork` 提示一次 VM fork 事件） |

## 2. 为什么 microVM 快照能恢复 netns 而 gVisor 不能

这是两种 C/R 哲学的差异，也是本次对照实验的价值所在：

1. **Firecracker snapshot = 物理机级冻结**。`PATCH /vm {"state":"Paused"}` 让所有 vCPU 退出 guest 模式，随后 `PUT /snapshot/create` 把每台 vCPU 的寄存器/`KVM_GET_VCPU_STATE`、guest 的**全部物理内存**逐页写入 `memory.snap`，把 virtio 设备（block/net）的队列状态和 VMM 侧元数据序列化进 `vmstate.snap`。guest kernel 自己维护的一切——路由表、netns 链表、bridge FDB、veth、conntrack、文件缓存——**全都活在那 4GB 物理内存镜像里**，restore 时把内存映射回去、vCPU 状态装回 KVM，guest kernel 根本"不知道"自己被冻结过。它没有任何"重建网络栈"的钩子，也就没有丢失的机会。
2. **gVisor C/R = 进程模型序列化**。sentry 在用户态模拟 kernel，checkpoint 只序列化它认识的对象：task 结构、内存页、文件描述符、**sentry 自己的 netstack**。而 sandbox 里 dockerd 通过 `AF_NETLINK` 发出的 `RTM_NEWLINK`（建 docker0/veth）最终作用在**宿主内核**为 sandbox 分配的真实 netns 上——这部分内核态对象不在 sentry 的对象图里，快照管不着；restore 后 sandbox 的网络设备表回到初始状态（lo + eth0），netlink 建的设备全部蒸发。
3. **推论**：进程级 C/R 天然有"内核态副作用覆盖不全"的边界（设备、cgroup、IPC、密钥环……每个都要单独建模）；VM 级快照用"全量内存 + 设备状态"换来了**语义完备性**，代价是镜像体积（GB 级 vs MB 级）和 restore 时对宿主 CPU 特性/内存布局的敏感性。这是 agent 沙箱 / serverless 平台选型时"隔离换状态完整性"的核心 trade-off。

## 3. 环境搭建全程（含踩坑）

### 3.1 编译 Firecracker（本机 rust 工具链）

- rustup 1.97.0 + **rsproxy.cn 镜像**（`RUSTUP_DIST_SERVER=https://rsproxy.cn/rustup`、`~/.cargo/config.toml` sparse index；`static.rust-lang.org` 直连只有 ~15KB/s，务必全换）。
- 目标 `x86_64-unknown-linux-musl`：`apt install musl-tools`；`userfaultfd-sys` crate 报缺 `linux/types.h` 时，把 UAPI 头链进 musl sysroot：
  ```bash
  ln -s /usr/include/linux /usr/include/x86_64-linux-musl/linux
  ln -s /usr/include/asm-generic /usr/include/x86_64-linux-musl/asm-generic
  ln -s /usr/include/x86_64-linux-gnu/asm /usr/include/x86_64-linux-musl/asm
  ```
- 产物路径（注意是 `build/cargo_target`，不是 `target/`）：
  `/home/keyang/Hypervisors/firecracker/build/cargo_target/x86_64-unknown-linux-musl/release/firecracker`
- /dev/kvm 属 root:kvm 660，当前用户不在 kvm 组 → firecracker 全程 `sudo` 运行。

### 3.2 guest 内核 / rootfs / initrd

- **内核**：Debian cloud 内核 `linux-image-6.12.88+deb12-cloud-amd64`（bzImage）。`CONFIG_VIRTIO_BLK/NET=m、BRIDGE=m、VETH=m、OVERLAY_FS=m` 都是模块；`virtio.ko/virtio_ring.ko` 内建。
- **rootfs**（10G ext4）：复用 gVisor 调研时的 docker-in-gvisor 镜像 `docker export` 解包（ubuntu 用户态 + docker-ce 28.5.2），植入 `/root/busybox.tar`、`/usr/bin/busybox-static`、`/fc-init.sh`（PID 1，无 systemd）。
- **initrd**（cpio.gz）：busybox:musl 静态版 + 完整模块树 + `/init`，专职加载 virtio 驱动后 `switch_root`。
- 宿主侧 tap：`fc-tap0` 172.20.0.1/24（VM eth0 为 172.20.0.2/24；本测试不依赖外网）。

### 3.3 踩坑记录（按时间序，全部已解决）

1. **Debian 模块是 `.ko.xz` 压缩的**，busybox 的 insmod 直接报 `Invalid ELF header magic: != ELF`。解决：整树 `xz -dk` 解压回裸 `.ko`，`depmod -b` 重生成索引（initrd 树和 rootfs 内各做一遍）。
2. **busybox-static 的 modprobe 是 MODPROBE_SMALL 小型实现**：不读 `modules.dep`、不解析依赖。表现为 `virtio_net: Unknown symbol net_failover_create (err -2)`。解决：initrd 和 fc-init.sh 全部改为**显式按依赖序 insmod**（`virtio_mmio → virtio_blk → failover → net_failover → virtio_net`；rootfs 侧 `llc → stp → bridge → veth → overlay`）。
3. **`virtio_mmio.ko` 不加载则设备根本不枚举**：内核命令行里 `virtio_mmio.device=4K@0xc0001000:5` 只是声明平台设备，总线驱动本身是模块。它不 insmod，`virtio_blk` 加载"成功"也永远没有 `/dev/vda`（且无任何报错）。必须最先加载。
4. **docker 导出的精简 ubuntu 镜像里没有 kmod**（无 `modprobe`/`insmod`），fc-init.sh 的 `insmod` 全部 "command not found"。解决：镜像里放 `/usr/bin/busybox-static` 并显式全路径调用。
5. **dockerd 启动时 bridge 模块未就绪会直接退出**：`failed to start daemon: Error initializing network controller: error creating default "bridge" network: operation not supported`。模块补齐后重启 dockerd 即恢复（正式流程已把模块加载放在 dockerd 之前）。
6. `/snapshot/load` **只接受 fresh 进程**：restore 前不得 PUT boot-source/drives/network-interfaces/machine-config（设备配置全在 vmstate.snap 里，包括 rootfs 磁盘路径和 guest MAC）。
7. 串口即控制台：firecracker 的 stdin/stdout 接 fifo + log 文件，向 `<name>.in` 写命令、从 `<name>.console.log` 读结果，即可脚本化驱动 guest（无需 ssh）。fifo 需要一个常驻写端（`sudo sleep infinity > fifo &`）防止 EOF。

### 3.4 关键脚本与 API 序列

启动（`/home/keyang/fc-cr-test/start-vm.sh`）：PUT `/boot-source`（bzImage+initrd+boot_args）→ `/drives/rootfs`（rw）→ `/network-interfaces/eth0`（tap `fc-tap0`，MAC `AA:FC:00:00:00:01`）→ `/machine-config`（4 vCPU / 4096MB / smt）→ `/actions InstanceStart`。

快照：
```bash
curl -X PATCH --unix-sock vm1.api.sock /vm -d '{"state":"Paused"}'          # 204
curl -X PUT  --unix-sock vm1.api.sock /snapshot/create -d '{"snapshot_path":".../vmstate.snap","mem_file_path":".../memory.snap"}'  # 204
```

恢复（新进程，不能预配置任何资源）：
```bash
start-vm.sh vm2 snapshot-load /home/keyang/fc-cr-test/snap1
# 即 PUT /snapshot/load {snapshot_path, mem_file_path, resume_vm:true}
```

## 4. 验证过程实录

### 4.1 Baseline（vm1，快照前）

```
CONTAINER ID   IMAGE     COMMAND                  STATUS          NAMES
82d855c1cc16   busybox   "sh -c 'i=0; while t…"   Up 28 seconds   stateful-app

lo            UNKNOWN  <LOOPBACK,UP,LOWER_UP>
eth0          UP       aa:fc:00:00:00:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
docker0       UP       5e:32:9a:3c:c3:4f <BROADCAST,MULTICAST,UP,LOWER_UP>   # 172.17.0.1/16
vethffe5324@if2 UP     e6:59:b5:b0:17:a6 <BROADCAST,MULTICAST,UP,LOWER_UP>

/var/run/docker/netns/: a731ac75c4ae
/tmp/count.log: tick 26/27/28（容器 PID 1892）
```

### 4.2 Pause → CreateSnapshot（~2s）

`memory.snap` 4.0G（= guest 全部物理内存）、`vmstate.snap` 44K。随后杀死 vm1。

### 4.3 Restore（vm2，~10ms 完成 load+resume）

```
API: {"state":"Running","vmm_version":"1.17.0-dev"}
guest: [181.938348] random: crng reseeded due to virtual machine fork   # 内核时钟从暂停点连续续走
uptime: up 3 min（连续）

docker ps:  stateful-app  Up About a minute（同一容器 ID 82d855c1cc16）
ip -br link: docker0（同 MAC 5e:32:9a:3c:c3:4f、172.17.0.1/16）、vethffe5324@if2（同 MAC）原样在
netns: /var/run/docker/netns/a731ac75c4ae 原样在
/tmp/count.log: tick 66→70 连续（1787216951→1787216953，暂停窗口仅 2s）
```

### 4.4 restore 后新建 bridge 容器 + 数据面（gVisor 侧在此失败）

```
docker run -d --name post-restore-app busybox ...   → 成功，新 veth veth74bd6fe@if2 挂上 docker0
docker exec post-restore-app ping -c2 172.17.0.1     → 0% loss（容器↔网关）
docker exec post-restore-app ping -c2 172.17.0.2     → 0% loss（新容器↔旧容器，跨快照）
docker network inspect bridge                        → stateful-app post-restore-app（同网）
```

## 5. 与 gVisor 方案的工程对比

| 维度 | gVisor runsc C/R | Firecracker snapshot |
|---|---|---|
| 快照语义 | 应用感知（task 对象图），需 runtime 逐类对象建模 | 物理机语义（内存+寄存器+设备），guest kernel 无感知 |
| 内核态副作用（netns/设备/cgroup） | 部分丢失（本次实测 netlink 设备全丢） | 完整保留 |
| 快照体积 | 与 sandbox 常驻内存相当（~200MB→59MB） | 与 VM 内存配置相等（4GB→4.0G，可用 diff snapshot 缓解） |
| restore 兼容性 | 受用户态工具链版本牵连（moby#50750） | 仅要求 CPU 特性一致（CPU template）+ 相同 firecracker 版本 |
| 冷启动速度 | 快照小、加载快 | 大页/UFFD 后台加载可优化；本例 load+resume 仅 ~10ms（page cache 已热） |
| 适用场景 | 与宿主共享内核、要求低内存开销 | 强隔离、状态完备性优先的沙箱/函数负载 |

## 6. 留档清单（`/home/keyang/fc-cr-test/`）

| 文件 | 说明 |
|---|---|
| `start-vm.sh` | 启动/恢复脚本（正常启动 & snapshot-load 两模式） |
| `initrd/`、`initrd.gz` | busybox initrd（解压后模块 + 显式依赖序 insmod） |
| `rootfs.ext4`（10G） | docker-in-gvisor 镜像导出 + 模块 + `/fc-init.sh`（已含全部修复） |
| `snap1/` | vmstate.snap（44K）+ memory.snap（4.0G） |
| `vm1.console.log` / `vm2.console.log` | 快照前 / 恢复后完整串口实录 |
| `kernel-extract/`、`linux-cloud.deb` | Debian cloud 内核与解包 |
| 编译产物 | `/home/keyang/Hypervisors/firecracker/build/cargo_target/x86_64-unknown-linux-musl/release/firecracker` |

测试后已清理：fc-tap0、firecracker 进程、串口 fifo。rootfs/ext4 与 snap1 保留，可直接重跑 restore。

## 7. 复现步骤（简版）

```bash
# 1. tap
sudo ip tuntap add fc-tap0 mode tap && sudo ip addr add 172.20.0.1/24 dev fc-tap0 && sudo ip link set fc-tap0 up
# 2. 冷启动（模块加载 → dockerd → busybox load → stateful-app → FC-DIND-READY → 串口 shell）
bash /home/keyang/fc-cr-test/start-vm.sh vm1
# 3. 快照
sudo curl -s --unix-socket vm1.api.sock -X PATCH .../vm -d '{"state":"Paused"}'
sudo curl -s --unix-socket vm1.api.sock -X PUT  .../snapshot/create -d '{...}'
# 4. 杀 vm1 后恢复
bash /home/keyang/fc-cr-test/start-vm.sh vm2 snapshot-load /home/keyang/fc-cr-test/snap1
# 5. 向 vm2.in 写命令验证：docker ps / ip -br link / 新建 bridge 容器 / ping
```
