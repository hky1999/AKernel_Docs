# AKernel 阿里云部署总结（cn-hongzhou / ACK）

> 本文档记录 2026-08-11 在阿里云 **cn-hangzhou** 通过 ACK + Terraform 成功部署 AKernel v0.1.1 的完整过程、最终可用配方，以及全程踩到的坑与解决办法。可作为后续在阿里云大陆地域复用的参考。

## 1. 部署结果

- **集群**：cn-hangzhou ACK 托管集群 `akernel`，K8s 1.34.3，flannel 网络，3 × `ecs.c7.3xlarge`
- **镜像**：全部托管在阿里云 ACR 个人版（仓库设为**公开**匿名拉取）
- **端到端验证通过**：`sdk/python/examples/basic_usage.py` 跑通——沙箱创建、命令执行、文件系统、后台命令、销毁全 OK
- **API 入口**：Traefik 公网 LoadBalancer（HTTPS:443 API/exec，HTTP:80 端口转发）

```
Sandbox created: 4e000000-...
Command output: Hello, AKernel!
File copy round trip: OK
Sandbox state: running
Sandbox terminated.
```

## 2. 适用场景与前置判断

- **目标**：在阿里云上跑一个可用的 AKernel 集群（runsc 沙箱）。
- **为什么选 cn-hangzhou（大陆地域）**：大陆地域的 ACK API（`cs.aliyuncs.com）从国内机器访问稳定；而香港/海外地域的 ACK API（`cs.cn-hongkong.aliyuncs.com`）跨境长连接会被反复 reset，导致 terraform 轮询集群创建状态时 EOF 失败。
- **大陆地域的代价**：节点**访问不了 Docker Hub**（`registry-1.docker.io` TCP 超时），所有镜像必须转推到阿里云 ACR。这是本部署方案的核心约束。

> 如果你有一台**稳定国际网络**的机器（代理 / VPN / 阿里云 Cloud Shell），可以改用 HK/海外地域，直接用 Docker Hub 公共镜像 `akerneldev/all-in-one`，省掉镜像转推步骤。本文不覆盖该路径。

## 3. 前置准备

### 3.1 阿里云账号
- 注册 + **实名认证**（否则开不了 ECS/ACK）
- **充值余额**：按量付费资源创建时会校验余额，余额不足报 `InvalidAccountStatus.NotEnoughBalance`。3 × c7.3xlarge + ACK Pro 大约 ¥7–10/h，建议至少充 ¥100。

### 3.2 RAM 子账号 + AccessKey
不要用主账号 AK。建一个 RAM 子账号（如 `akernel-deploy`），并：
- 生成 **OpenAPI AccessKey**：ID 形如 `LTAI5t...`（**24 位、以 `LTAI` 开头**），Secret 只显示一次。
- 授权 **`AdministratorAccess`**，且**授权范围必须是「整个云账号」/账号级**——资源组级别的策略会导致 ACK 创建过程中 `Forbidden.RAM` / `ImplicitDeny`（如 `vpc:DescribeNatGateways` 被拒）。

凭证存到仓库外（gitignored），例如 `/mnt/u/hukeyang/AKernel/.alicloud_rc`：
```bash
export ALICLOUD_ACCESS_KEY="LTAI5t..."
export ALICLOUD_SECRET_KEY="<secret>"
export ALICLOUD_REGION="cn-hangzhou"
```

### 3.3 一次性 RAM 角色授权（控制台）
全新账号第一次用 API/Terraform 建 ACK 会缺角色，必须先在控制台授权（API 不会自动触发）：
- **ACK 默认角色**：缺少时报 `EntityNotExist.Role: aliyuncsdefaultrole`。打开 ACK 控制台，按提示一键授权。
- **OOS 生命周期角色**：缺少时报 `MissingAuth.AliyunOOSLifecycleHook4CSRole`。错误信息里会带一个 `ram.console.aliyun.com/role/authorize?request=...` 链接，点开确认授权即可（terraform 自动创建的同名角色不被 ACK 认）。

### 3.4 本地工具链
```bash
make check VENDOR=aliyun
```
缺什么装什么：`terraform >= 1.5`、`helm`、`kubectl`、`docker`、`git submodules`。
```bash
git submodule update --init --recursive   # 必须先初始化
```

### 3.5 阿里云容器镜像服务 ACR（个人版）
- 创建一个 **Personal 实例**（cn-hangzhou），命名空间如 `akerneldev`。
- 创建仓库 `akerneldev`（即 `akerneldev/akerneldev`）。
- **仓库类型设为「公开」**——个人版私有仓库节点拉取需要 imagePullSecret，而 terraform 模块未暴露 imagePullSecret 变量；公开后匿名拉取最省事。
- 记下推送地址（公网）：`crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com`

## 4. 镜像准备（转推到 ACR）

cn-hangzhou 节点拉不到 Docker Hub，本机构建机能（跨境网络偶发抖动，带重试）。把以下镜像拉到本地再推到 ACR（同一仓库、不同 tag）：

| 镜像 | 来源 | 推送到 ACR 的 tag |
|---|---|---|
| `akerneldev/all-in-one:latest` | Docker Hub（~9.45GB） | `latest` |
| `traefik:v3.6.8` | Docker Hub | `traefik-v3.6.8` |
| etcd（自建） | 见下 | `etcd-v3.6.8` |

### 4.1 etcd 的特殊处理（关键坑）
chart 的 etcd 模板需要镜像**带 shell**（命令用 `/bin/sh -ec "mkdir...exec etcd ..."` 包裹）：
- chart 默认 `public.ecr.aws/bitnami/etcd:3.6.8` —— bitnami 已从 Docker Hub 下架，ECR public 从国内和阿里云节点都拉不到（404）。
- 官方 `gcr.io/etcd-development/etcd:v3.6.8` —— **distroless，无 `/bin/sh`**，直接用会失败。
- **解法**：用 gcr 镜像里的 etcd 二进制（静态链接）+ alpine 自建一个带 shell 的镜像：

```bash
# 拉取（重试，跨境抖动）
docker pull gcr.io/etcd-development/etcd:v3.6.8
# 取出二进制
cid=$(docker create gcr.io/etcd-development/etcd:v3.6.8 /usr/local/bin/etcd)
docker cp "$cid:/usr/local/bin/" /tmp/etcd-bin/
docker rm "$cid" >/dev/null

# 构建带 shell 的 etcd 镜像
cd /tmp/etcd-bin && cat > Dockerfile <<'EOF'
FROM alpine:3.20
RUN apk add --no-cache ca-certificates libc6-compat
COPY etcd etcdctl etcdutl /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/etcd"]
EOF
docker build -t etcd-shell:v3.6.8 .
docker run --rm --entrypoint /bin/sh etcd-shell:v3.6.8 -c 'etcd --version'  # 验证
```

### 4.2 推送到 ACR
> ⚠️ **zsh 坑**：本仓库 shell 是 zsh，`$ACR:tag` 会被 `:t`/`:l` 等路径修饰符静默损坏（如 `$ACR:traefik` → basename `akerneldev`）。**必须用 `${ACR}:tag` 大括号语法。**

```bash
ACR="crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev/akerneldev"
docker login --username=<账号> crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com

docker tag akerneldev/all-in-one:latest "${ACR}:latest"
docker tag traefik:v3.6.8          "${ACR}:traefik-v3.6.8"
docker tag etcd-shell:v3.6.8       "${ACR}:etcd-v3.6.8"

docker push "${ACR}:latest"
docker push "${ACR}:traefik-v3.6.8"
docker push "${ACR}:etcd-v3.6.8"
```

拉取上游镜像时若遇 EOF，用重试循环（分层缓存，重试会逐渐吃下）：
```bash
for i in $(seq 1 15); do docker pull akerneldev/all-in-one:latest && break; sleep 3; done
```

## 5. 配置与部署

### 5.1 生成配置
```bash
make config \
  ENV=default VENDOR=aliyun NON_INTERACTIVE=1 FORCE=1 \
  REGION=cn-hangzhou \
  ZONE_IDS="cn-hangzhou-j,cn-hangzhou-j,cn-hangzhou-j" \
  IMAGE_REPOSITORY="crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev/akerneldev" \
  IMAGE_TAG="latest" \
  INSTALL_MONITOR=false
```
- 用默认机型 `ecs.c7.3xlarge` × 3（`make config` 默认值）。
- `INSTALL_MONITOR=false`：monitor chart 的镜像（grafana/prom/loki/tempo/busybox）全在 Docker Hub，大陆节点拉不到，要么再转推 5 个镜像，要么直接关。

### 5.2 修补 tfvars（configure.sh 未暴露的关键项）
`.akernel/default/terraform.tfvars` 里改/补：

```hcl
# 1) pod 密度修复：terway-eniip 在 c7/g7 上只有 6 pods/node，集群组件挤不下
network_addon = "flannel"

# 2) traefik 镜像走 ACR
traefik_image_repository = "crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev/akerneldev"
traefik_image_tag        = "traefik-v3.6.8"

# 3) etcd 镜像走 ACR（自建的带 shell 版本）
etcd_image_repository = "crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev/akerneldev"
etcd_image_tag        = "etcd-v3.6.8"

# 4) 关掉 traefik 非关键 sidecar：它拉 registry-vpc.../akernel/busybox（不存在的 repo，403）
traefik_internal_stats_enabled = false
```
> `master_image_repository` / `node_image_repository` 已由 `IMAGE_REPOSITORY` 设为 ACR。`k8s_version` 默认 `1.34.3-aliyun.1` 在 cn-hangzhou 可用（HK 需要 1.34.10+）。

### 5.3 部署
```bash
set -a; source /mnt/u/hukeyang/AKernel/.alicloud_rc; set +a
export AUTO_APPROVE=1
make plan     # 审查
make deploy   # 实际创建云资源，计费开始
```
`make deploy` 结束会打印 `AKERNEL_SERVER_ADDRESS` 和一个 SDK token。

### 5.4 验证
```bash
# Pod 全部 1/1 Running
KUBECONFIG=.akernel/default/kubeconfig kubectl -n akernel get pods

# 端到端
python3 -m pip install -e ./sdk/python --break-system-packages   # 首次需要
export AKERNEL_SERVER_ADDRESS=<deploy 输出的地址>
export AKERNEL_TOKEN="$(make token TTL=24h 2>/dev/null | sed -n "s/^export AKERNEL_TOKEN='\(.*\)'/\1/p")"
python3 ./sdk/python/examples/basic_usage.py
```

## 6. 关键设计决策（为什么这么配）

| 决策 | 原因 |
|---|---|
| `network_addon=flannel` | terway-eniip 在 c7/g7 上每节点仅 6 个 pod 配额，系统组件占满后 AKernel 组件全部 `Too many pods` 调度失败。flannel 用 pod CIDR，~110 pods/node。terway 模式集群创建后不可改，必须建集群时就选 flannel。 |
| 镜像走 ACR + 仓库公开 | cn-hangzhou 节点访问不了 Docker Hub；私有 ACR 需 imagePullSecret 但模块未暴露变量；公开匿名拉取最简。 |
| 自建 etcd 镜像 | chart 需要带 shell 的 etcd；bitnami 不可达，官方 gcr 是 distroless。 |
| 关闭 monitor | 其镜像全在 Docker Hub，转推成本高，验证 AKernel 本身不需要。 |
| 关闭 traefik internalStats | sidecar 镜像被模块改写为不存在的阿里云 busybox repo（403）；非关键功能。 |

## 7. 踩坑全记录（按出现顺序）

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 1 | `InvalidAccessKeyId.NotFound` | AK ID 复制漏了首字母 `L`（应为 `LTAI...` 24 位） | 重新完整复制 |
| 2 | `Forbidden.RAM` / `ImplicitDeny` (vpc:DescribeNatGateways) | RAM 子账号权限是**资源组级** | 改授账号级 `AdministratorAccess` |
| 3 | `EntityNotExist.Role: aliyuncsdefaultrole` | 新账号缺 ACK 默认角色 | ACK 控制台首次使用一键授权 |
| 4 | `MissingAuth.AliyunOOSLifecycleHook4CSRole` | OOS 角色未授权（terraform 自建的不被认） | 点错误信息里的 `ram.console.aliyun.com/role/authorize` 链接授权 |
| 5 | `InvalidAccountStatus.NotEnoughBalance` | 账户余额不足，按量 ECS 下单被拒 | 充值 |
| 6 | `InvalidKubernetesVersion` (HK) | configure.sh 写死 `1.34.3`，HK 只认 `1.34.10+` | tfvars 改 `k8s_version`（cn-hangzhou 不受影响） |
| 7 | HK ACK 创建反复 EOF | 跨境长连接被 reset，terraform 轮询失败 | 放弃 HK，回 cn-hangzhou |
| 8 | registry.terraform.io EOF | 同跨境抖动 | 多为偶发，重试或用 provider 本地 mirror |
| 9 | Pod 全部 ImagePullBackOff (docker.io) | cn-hangzhou 节点连不上 Docker Hub | 镜像转推 ACR |
| 10 | 6 pods/node，组件调度不下 | terway-eniip ENI 限制 | 改 flannel |
| 11 | `pull access denied ... insufficient_scope` (节点拉 ACR) | 仓库私有，节点无凭证 | ACR 仓库设公开 |
| 12 | etcd `Init:ImagePullBackOff` | bitnami/etcd 不可达 | 自建 alpine+etcd 镜像推 ACR |
| 13 | traefik sidecar 403 | internalStats 拉不存在的 busybox repo | `traefik_internal_stats_enabled=false` |
| 14 | docker push 推到 `docker.io/library/akerneldevraefik-...` | zsh `$ACR:traefik` 被 `:t` 修饰符取 basename | 用 `${ACR}:tag` |

## 8. 成本与清理

集群**持续计费**（约 ¥7–10/h）。验证完毕及时销毁：
```bash
set -a; source /mnt/u/hukeyang/AKernel/.alicloud_rc; set +a
export AUTO_APPROVE=1
make destroy ENV=default
```
`.akernel/default/` 配置/state 会保留，下次 `make deploy` 可直接重建（iam-seed 复用，旧 token 仍兼容）。

> 也可保留集群，用 `make token TTL=24h` 随时生成本地 SDK 凭证连接使用。

## 9. 日常使用速查

```bash
# 连接信息
export AKERNEL_SERVER_ADDRESS=<地址>      # make print-env 查看
export AKERNEL_TOKEN="$(make token TTL=24h ...)"  # 本地签发，无需暴露 IAM API

# Python SDK
from akernel_sdk import Sandbox
with Sandbox(cpu=1000, memory=2048) as sb:
    print(sb.commands.run("echo hello").stdout)

# 排查
KUBECONFIG=.akernel/default/kubeconfig kubectl -n akernel get pods
docker logs ... # 节点内组件
```

## 10. 未覆盖 / 待办

- **Kata 运行时**：普通 ECS 不暴露 `/dev/kvm`，本部署只有 runsc。需要 Kata 要用支持嵌套虚拟化的实例。
- **cgroup v1 要求**：v0.1.0/v0.1.1 node 运行时要求 cgroup v1（见 CLAUDE.md）；Alibaba Linux 3 默认 cgroup v2，需确认节点镜像。
- **monitor 栈**：默认关闭。开启方法见第 11 节（需转推 5 个镜像 + 公开对应 ACR repo）。
- **HK/海外地域**：需要稳定国际网络环境（代理/Cloud Shell），否则 ACK 创建会被跨境 EOF 卡死。

## 11. 可选：开启监控栈（Grafana / Prometheus / Loki / Tempo）

基础部署（第 4–5 节）默认 `install_monitor=false`。开启后会额外安装 Grafana 大盘（带公网 LB）+ Prometheus 指标 + Loki 日志 + Tempo 链路。monitor 镜像全部在 Docker Hub，必须像核心镜像一样转推到 ACR。

### 11.1 转推 5 个 monitor 镜像

镜像与目标路径（`monitor_image_registry` 作为前缀，模板拼成 `${registry}/<name>`，即 namespace 下每个组件一个独立 repo）：

| 来源 | 推送到 ACR |
|---|---|
| `busybox:1.37.0-musl` | `akerneldev/busybox:1.37.0-musl` |
| `grafana/grafana:11.5.2` | `akerneldev/grafana:11.5.2` |
| `prom/prometheus:v3.2.1` | `akerneldev/prometheus:v3.2.1` |
| `grafana/loki:3.4.2` | `akerneldev/loki:3.4.2` |
| `grafana/tempo:2.7.1` | `akerneldev/tempo:2.7.1` |

```bash
ACR_NS="crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev"
docker login --username=<账号> crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com
# 用 ${ACR_NS} 大括号防 zsh 修饰符（见第 7 节坑 14）
docker tag busybox:1.37.0-musl      "${ACR_NS}/busybox:1.37.0-musl";    docker push "${ACR_NS}/busybox:1.37.0-musl"
docker tag grafana/grafana:11.5.2   "${ACR_NS}/grafana:11.5.2";         docker push "${ACR_NS}/grafana:11.5.2"
docker tag prom/prometheus:v3.2.1   "${ACR_NS}/prometheus:v3.2.1";      docker push "${ACR_NS}/prometheus:v3.2.1"
docker tag grafana/tempo:2.7.1      "${ACR_NS}/tempo:2.7.1";            docker push "${ACR_NS}/tempo:2.7.1"
# loki 见 11.2 特殊处理
```

> ⚠️ 这 5 个 repo 是 push 时**自动创建**的，ACR 个人版默认**私有**。必须到控制台把这 5 个 repo（`grafana`/`prometheus`/`loki`/`tempo`/`busybox`）都改成**公开**，否则节点匿名拉取报 `insufficient_scope`（同核心镜像的坑）。

### 11.2 loki 特殊处理（多架构镜像坑）

`grafana/loki:3.4.2` 是多架构 manifest list，直接 `docker push` 到 ACR 报：
```
image with reference ... was found but does not provide any platform
```
（traefik 等其他镜像会自动以单平台 push，唯独 loki 不行）。用 `export|import` 拍平成单平台镜像，保留 entrypoint（chart 依赖 `/usr/bin/loki`）：

```bash
docker rm -f lokitmp 2>/dev/null
docker create --name lokitmp grafana/loki:3.4.2 >/dev/null
docker export lokitmp | docker import \
  --change 'ENTRYPOINT ["/usr/bin/loki"]' \
  --change 'ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt' \
  --change 'ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/busybox' \
  - "${ACR_NS}/loki:3.4.2"
docker rm lokitmp >/dev/null
docker push "${ACR_NS}/loki:3.4.2"
```
（chart 的 loki deployment 自带 `args: -config.file=...` 和 config volumeMount，所以拍平丢 CMD 不影响。）

### 11.3 开启 monitor 并部署

`.akernel/default/terraform.tfvars`：
```hcl
install_monitor       = true
grafana_public_access = true
monitor_image_registry = "crpi-<instance>.cn-hangzhou.personal.cr.aliyuncs.com/akerneldev"
```
然后 `make deploy`（helm 会升级 core 并新装 `akernel-monitor` 子 chart）。

### 11.4 registry.terraform.io init 偶发 EOF（跨境抖动）

部署/升级时 `terraform init` 可能因跨境网络抖动报：
```
could not connect to registry.terraform.io: ... unexpected EOF
```
providers 其实已缓存在 `.akernel/terraform-plugin-cache/`。配一个本地 mirror 让 init 不碰 registry。`~/.terraformrc`：
```
provider_installation {
  filesystem_mirror {
    path    = "/mnt/u/hukeyang/AKernel/AKernel/.akernel/terraform-plugin-cache"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

### 11.5 Grafana 访问

部署完成后 `make deploy` / `make print-env` 会打印 Grafana 地址，也可：
```bash
kubectl -n akernel-monitor get svc grafana   # EXTERNAL-IP:3000
```
- URL：`http://<grafana-LB-IP>:3000`
- 用户名：`admin`
- 密码：`.akernel/default/grafana-admin-password`

> ⚠️ Grafana 是**公网 LB + 默认随机密码**，注意安全：及时改密码，或用 `grafana_acl_id` 给 LB 加访问白名单；不用就 `make destroy`。

### 11.6 开启 monitor 的额外成本

- 1 个 Grafana 公网 SLB
- Grafana / Prometheus / Loki 各 1 块 ESSD PVC（默认各 ~20Gi，见 `*_pvc_size`）
- 4 个 monitor Pod 的计算资源（见 `grafana_resources`/`prometheus_resources`/`loki_resources`/`tempo_resources`）
估算额外 ¥1–2/h。
