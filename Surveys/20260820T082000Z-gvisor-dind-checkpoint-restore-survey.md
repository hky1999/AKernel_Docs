# gVisor 内运行 Docker（DinD）并做 Checkpoint/Restore：复现、踩坑与验证

> 调研时间：2026-08-20（Asia/Shanghai）
> 文档时间戳：20260820T082000Z
> 依据文档：gVisor 官方教程 <https://gvisor.dev/docs/tutorials/docker-in-gvisor/>、<https://gvisor.dev/docs/user_guide/checkpoint_restore/>
> 源码仓库：`/home/keyang/AKernelWorkspace/gvisor-stock`（上游 google/gvisor，HEAD `a29997e6c`，runsc 版本 `release-20260706.0-dirty`）
> 方法：本机（ubuntu-vm，Linux 5.15，47G 内存）搭建**完全隔离**的测试环境实测；不动系统 dockerd（其上有运行 9 天的容器）与任何远程服务器。

## 1. 结论摘要

**结论：可行。** 在 runsc sandbox 里可以跑起完整 Docker（dockerd 28.5.2 + containerd + runc），并对**外层 gVisor 容器**做 checkpoint/restore。restore 之后：

| 验证项 | 结果 |
|---|---|
| 外层容器（gVisor sandbox）恢复运行 | ✅ `docker start --checkpoint` 后 Up |
| 内层 dockerd 存活、API 可用 | ✅ `docker ps` / `docker exec` / `docker load` 正常 |
| 内层业务容器进程状态（内存、计数器、文件内容） | ✅ 计数器从 checkpoint 时刻**无缝继续**，未重置 |
| 内层容器的**运行数据**（写入的文件） | ✅ 完整保留 |
| restore 后新建内层容器（`--network none`） | ✅ 正常 |
| restore 后新建内层容器（默认 bridge 网络） | ❌ 失败：sandbox 内运行时创建的 `docker0` 网桥不随 C/R 恢复 |
| 内层容器自身网络（veth/eth0） | ❌ restore 后 netns 内只剩 `lo`，veth 丢失（进程不受影响） |
| C/R 可重复性 | ✅ 连续两轮 C/R，计数器全程连续（tick 6 → 55） |
| C/R 开销（本例，sandbox 常驻内存 ~200MB 级） | checkpoint ≈ 2.2~2.6s，restore ≈ 1.9s，快照 59MB（pages.img 58MB + checkpoint.img 1MB） |

**关键前置坑（按踩中顺序）：**

1. **Docker ≥ 28 的 `docker start --checkpoint` 是坏的**（moby#50750）。本机 Ubuntu 打包的 docker 29.1.3 上 checkpoint 能创建、restore 必报 `bind-mount /proc/0/ns/net ...: no such file or directory`。gVisor 自己的 e2e 测试代码在 docker ≥ 28 时直接 skip restore（`pkg/test/dockerutil/dockerutil.go:131`）。**必须用 docker ≤ 27 做外层 daemon**（本文用 docker-ce 27.5.1 + containerd.io 1.7.27）。
2. containerd 1.7 + 系统自带的 containerd 2.2 shim 会报 `unsupported shim version (3): not implemented`，需保证私有栈 PATH 里 1.7 自带的 shim 优先。
3. 内层普通容器 `docker run` 会因 runc masked-paths 挂 `/sys/firmware` 失败（gVisor tmpfs 不支持 `nr_blocks=1` 选项），内层容器需要 `--privileged`。
4. `download.docker.com` 被墙，构建 docker-in-gvisor 镜像需换国内 apt 源；`registry-1.docker.io` 不通，需走镜像源或预导入镜像 tar。

## 2. 源码依据

### 2.1 教程对应的上游实现（gvisor-stock）

| 教程要求 | 源码/文件 |
|---|---|
| `--net-raw`（Docker 所有版本必需）、`--allow-packet-socket-write`（v28+ 必需，dockerd v28+ 起网口时发 unsolicited ARP 需要 AF_PACKET 写） | 教程 Configuration Reference |
| `/var/lib/docker` 挂 tmpfs（gVisor 的 overlayfs 只支持 tmpfs 作 upper layer），或 `--feature containerd-snapshotter=false` | `gvisor-stock/images/basic/docker/start-dockerd.sh`（本仓库自带该参考实现） |
| dockerd 需 `--iptables=false --ip6tables=false` | 同上 |
| Docker-in-gVisor 镜像 | `gvisor-stock/images/basic/docker/Dockerfile`（ubuntu 24.04 + docker-ce 28.5.2） |
| C/R 的 Docker 流程 `docker checkpoint create` → `docker start --checkpoint` | 官方 checkpoint_restore 文档；e2e 测试 `test/e2e/integration_runtime_test.go:248` `TestOverlayCheckpointRestore` |

### 2.2 "Docker ≥ 28 restore 坏了"的根因链（本机实测 + 源码定位）

1. **gVisor 侧的证据**（`pkg/test/dockerutil/dockerutil.go`）：

   ```go
   // IsRestoreSupported returns true if the docker version supports restore.
   // Docker restore broke starting v28 due to
   // https://github.com/moby/moby/issues/50750.
   func IsRestoreSupported() bool {
       major, _ := getDockerVersion()
       return major <= 27
   }
   ```

2. **moby 侧**：restore 时 `containerStart` 的顺序是 `NewTask(checkpoint)` → `initializeCreatedTask` → `tsk.Start()`（moby master `daemon/start.go:226-253`）。`initializeCreatedTask`（`daemon/start_linux.go:33`）做网络初始化：

   ```go
   if err := sb.SetKey(ctx, fmt.Sprintf("/proc/%d/ns/net", tsk.Pid())); err != nil { ... }
   ```

3. **containerd 侧**（v2.2.1 `cmd/containerd-shim-runc-v2/process/init.go:135`）：带 checkpoint 的 Create 走 `createCheckpointedState()`，只**记录** RestoreOpts（`initState = &createdCheckpointState{}`），真正的 `runtime restore` 推迟到 Start。因此 Create 返回的 pid = 0。

4. 两者叠加：moby 拿 pid 0 拼 `/proc/0/ns/net` → ENOENT → start 失败 → cleanup 删容器。runsc 自始至终没被调用（runsc debug 日志里只有 pause/checkpoint/resume/delete，无 restore）。**该 bug 与 gVisor/runsc 无关，对任何 runtime 都一样。**

   实测佐证：docker 29.1.3 上同样的命令序列，checkpoint create 成功（2.2s），restore 100% 复现失败；换成 docker-ce 27.5.1 + containerd 1.7.27 后同一流程立即成功。

### 2.3 containerd 1.7 的 shim 版本坑

containerd 1.7 通过 PATH 查找 `containerd-shim-runc-v2`。本机系统 shim 是 containerd 2.2.1 的（协议 v3），1.7 只认 v2，报：

```text
failed to create task for container: Unimplemented: failed to start shim:
start failed: unsupported shim version (3): not implemented
```

（containerd 1.7.27 `runtime/v2/shim.go:221`：`params.Version > 2 → not implemented`）
解法：启动私有 containerd/dockerd 时把 `docker27/rootfs27/usr/bin` 放 PATH 最前。

## 3. 测试环境搭建（完全隔离，不影响宿主）

### 3.1 背景

- 系统 dockerd（29.1.3）不能加 runtime 配置（需重启，会打断别人跑了 9 天的容器），因此起了私有 dockerd：独立 socket、`data-root`、`exec-root`、pidfile。
- 系统 dockerd 连的是系统 containerd（2.2.1，`/run/containerd/containerd.sock`）；dockerd 默认**会复用已存在的 /run/containerd socket**，所以私有栈必须显式 `--containerd` 指到自己的 containerd 1.7.27。
- 网络面只做加法：手动建一个空 bridge `gvisorcr0`，dockerd 配 `"iptables": false`（不动宿主 iptables），外层容器走该 bridge；不需要对外连通（题目明确）。

### 3.2 材料

| 组件 | 版本 | 来源 |
|---|---|---|
| runsc | `release-20260706.0-dirty`（systrap 平台，静态编译） | `gvisor-stock/bazel-bin/runsc/runsc_/runsc`（仓库内预构建产物，拷贝到 `~/gvisor-cr-test/runsc`） |
| 外层 dockerd（最终成功配置） | docker-ce 27.5.1 | `mirrors.aliyun.com/docker-ce/linux/ubuntu`（noble pool），dpkg-deb -x 解出 |
| containerd | containerd.io 1.7.27 | 同上 |
| docker-in-gvisor 镜像 | ubuntu 24.04 + docker-ce 28.5.2 | `gvisor-stock/images/basic/docker/Dockerfile`（apt 源换成 aliyun docker-ce 镜像，其余一致） |
| 内层业务镜像 | busybox | 宿主 `docker save` 后 bind-mount 进去 `docker load` |

目录：`/home/keyang/gvisor-cr-test/`（daemon27.json、shared/、runsc-logs/、dockerd27.log、containerd17.log 等全部留档）。

### 3.3 关键命令

私有 daemon27.json（要点：runsc runtime + `--net-raw --allow-packet-socket-write` + 独立 bridge + iptables 关闭 + experimental）：

```json
{
  "data-root": "/home/keyang/gvisor-cr-test/docker-data-27",
  "exec-root": "/home/keyang/gvisor-cr-test/docker-exec-27",
  "pidfile": "/home/keyang/gvisor-cr-test/dockerd27.pid",
  "experimental": true,
  "iptables": false, "ip6tables": false,
  "bridge": "gvisorcr0",
  "default-address-pools": [{ "base": "172.31.201.0/24", "size": 24 }],
  "runtimes": {
    "runsc": {
      "path": "/home/keyang/gvisor-cr-test/runsc",
      "runtimeArgs": ["--net-raw", "--allow-packet-socket-write",
                      "--debug-log=/home/keyang/gvisor-cr-test/runsc-logs/"]
    }
  }
}
```

起私有栈（注意 PATH 让 1.7 的 shim 优先）：

```bash
sudo ip link add gvisorcr0 type bridge && sudo ip link set gvisorcr0 up
sudo setsid env PATH=$T/docker27/rootfs27/usr/bin:$PATH \
  $T/docker27/rootfs27/usr/bin/containerd \
  -a $T/containerd17.sock --root $T/containerd17-root --state $T/containerd17-state &
sudo setsid env PATH=$T/docker27/rootfs27/usr/bin:$PATH \
  $T/docker27/rootfs27/usr/bin/dockerd \
  --containerd $T/containerd17.sock --config-file $T/daemon27.json \
  -H unix://$T/docker27.sock &
```

外层容器 entrypoint 用了 `start-dockerd-nonet.sh`（基于上游 `start-dockerd.sh` 的变体：保留 tmpfs-on-/var/lib/docker 与 `--iptables=false --ip6tables=false`，去掉 SNAT/默认路由部分——原脚本在无外网需求场景下会因取不到默认路由设备而退出；留档于 `gvisor-cr-test/shared/`）：

```bash
docker run --runtime=runsc -d --name docker-in-gvisor --cap-add all \
  --entrypoint /shared/start-dockerd-nonet.sh \
  -v $T/shared:/shared docker-in-gvisor
# 进容器后：
docker load -i /shared/busybox.tar
docker run -d --privileged --name stateful-app busybox sh -c \
  'i=0; while true; do echo "tick $i $(date +%s)" >> /tmp/count.log; sleep 1; i=$((i+1)); done'
```

## 4. C/R 实测记录

### 4.1 第一轮（外层 docker 29.1.3，失败，定位过程）

| 步骤 | 结果 |
|---|---|
| 内层 dockerd 起来、stateful-app 跑起来 | ✅（tick 递增） |
| `docker checkpoint create docker-in-gvisor cp1` | ✅ 2.2s，容器按预期 Exited(0) |
| `docker start --checkpoint cp1 docker-in-gvisor`（`--network none` 和 bridge 两种外层网络都试了） | ❌ `bind-mount /proc/0/ns/net -> docker-exec/netns/xxx: no such file or directory`，100% 复现 |

定位：dockerd debug 日志显示 `/tasks/create` 事件已出（pid=0）→ moby 在 Create 与 Start 之间做网络初始化用 `/proc/0/ns/net` → 失败；runsc debug 日志（`runsc-logs/`）显示 restore 阶段 runsc 根本未被调用。根因链见 §2.2。

### 4.2 第二轮（外层 docker-ce 27.5.1 + containerd 1.7.27，成功）

```text
=== BEFORE checkpoint ===
stateful-app | Up 6 seconds
tick 6 1787213557                      ← checkpoint 前最后一条

=== checkpoint create cp1 ===           2.24s，容器 Exited(0)
=== restore (docker start --checkpoint cp1) ===  1.90s，容器 Up

=== AFTER restore ===
stateful-app | Up 31 seconds
tick 23 1787213579
tick 26 1787213582                     ← 计数器未重置，从 checkpoint 时刻继续
（再等 5s）tick 31 1787213587           ← 持续运行
```

功能验证：

```text
restore 后 docker exec stateful-app tail /tmp/count.log   ✅（内层 dockerd exec 管道完整）
restore 后 docker run --rm --network none busybox echo OK  ✅（能新建容器）
restore 后 docker run --rm busybox（默认 bridge）           ❌ adding veth to docker0: Device does not exist
restore 后 sandbox 内 ip link                              只剩 lo + eth0，内层 docker0/veth 均未恢复
```

第二轮 C/R（cp2）：checkpoint 2.57s / restore 1.94s，计数 tick 48 → 55 连续。

checkpoint 产物（`docker-data-27/containers/<id>/checkpoints/cp1/`）：

```text
checkpoint.img  1.0MB   pages.img  58MB   pages_meta.img  34KB    （合计 59MB）
```

### 4.3 网络不恢复的影响面评估

- **进程/内存/文件系统状态完全保留**：内层 dockerd、containerd-shim、runc、业务进程全部原地继续；写盘数据（count.log）无丢失。
- **sandbox 内运行时创建的网络设备（内层 docker0、各容器 veth）不恢复**：已有容器进程无恙但其 netns 只剩 `lo`；依赖内层 bridge 的新容器起不来。这不影响"状态保活/迁移"类场景（本题关注点），但要注意：若内层业务在 restore 后需要新建容器或网络通信，需要重启内层 dockerd 或改 `--network none/host`。
- gVisor 官方文档声明 C/R 支持 `--network=sandbox`（默认）指的是**外层 sandbox 自己的 netstack 状态**（eth0 配置等），与"sandbox 内进程再创建的 netlink 设备"是两回事——后者目前不在恢复范围内（本文实测）。

## 5. 复现步骤清单（本机可直接照抄）

1. `sudo docker build --network=host -t docker-in-gvisor /home/keyang/gvisor-cr-test/build/`（用 CN 源 Dockerfile；系统 dockerd 没有 docker0，build 必须走 host 网络）
2. `sudo docker save busybox -o $T/shared/busybox.tar`；`docker-in-gvisor` 同样 save 后 `docker -H unix://$T/docker27.sock load`
3. 按 §3.3 起私有 containerd 1.7 + dockerd 27（`gvisorcr0` bridge 需先手动创建）
4. 起外层容器、load busybox、起 stateful-app
5. `docker checkpoint create docker-in-gvisor cp1` → `docker start --checkpoint cp1 docker-in-gvisor`
6. 验证：内层 `docker ps`、`docker exec stateful-app tail /tmp/count.log`（看计数连续性）
7. 清理：`docker rm -f`、停 dockerd27/containerd17、`sudo ip link del gvisorcr0`、按需删 `$T/docker-data-27`

## 6. 对 AKernel 的启示

- **gVisor DinD 整体可 C/R**：外层 sandbox 一个 checkpoint 就把"dockerd + 全部内层容器进程 + 内存 + tmpfs 数据"打包带走（59MB / ~2s 级），对"带环境的 agent 工作现场快照/迁移"是可行路径；内层容器无感知、无需逐个 checkpoint。
- **真正的坑在 Docker 生态而非 gVisor**：`docker start --checkpoint` 在 moby ≥ 28 回归损坏（moby#50750），且上游短期不修（gVisor e2e 都在绕）。任何要走 Docker API 做 C/R 的链路，外层 daemon 目前必须钉在 docker ≤ 27；若 AKernel 自己控制 OCI/runsc 调用链（不走 docker CLI），则不受此影响——gVisor 文档也把 raw `runsc checkpoint/restore` 列为首选接口。
- **网络面是 C/R 的边界**：sandbox 内运行时创建的网桥/veth 不恢复。AKernel 若在 sandbox 里跑 DinD 或其它自组网组件，restore 后需要网络自愈逻辑（或全部走 none/host 网络）。
- 内层 runc 需要 `--privileged` 才能在 gVisor 里起容器（masked-paths 的 tmpfs `nr_blocks` 选项 gVisor 不支持）；gVisor 官方也说明 `--cap-add/--privileged` 只影响 sandbox 内观感，不给宿主权限。

## 附：文件留档索引

| 内容 | 路径 |
|---|---|
| 测试根目录 | `/home/keyang/gvisor-cr-test/` |
| daemon 配置（27 成功版 / 29 失败版） | `daemon27.json` / `daemon.json` |
| 外层 entrypoint 变体 | `shared/start-dockerd-nonet.sh` |
| 镜像构建目录（CN 源 Dockerfile） | `build/` |
| docker27/containerd 1.7 解包 | `docker27/rootfs27/` |
| runsc debug 日志（含 pause/checkpoint/resume/delete 全序列） | `runsc-logs/` |
| dockerd/containerd 日志 | `dockerd27.log`、`containerd17.log`、`dockerd.log` |
| checkpoint 镜像 | `docker-data-27/containers/<id>/checkpoints/{cp1,cp2}/` |
