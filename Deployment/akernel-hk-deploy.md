# AKernel 部署总结 —— 阿里云香港 (cn-hongkong)

> 基于 AKernel **v0.1.1**,部署到阿里云 ACK(香港区),使用 DockerHub 公共镜像 `akerneldev/all-in-one:v0.1.1`,无需自建/推送镜像。本文记录完整的部署方法、踩坑过程、验证方式与大盘查看。

---

## 0. 最终部署信息速查

| 项 | 值 |
|---|---|
| 版本 | AKernel `v0.1.1` |
| 云厂商 / 区域 | 阿里云 `cn-hongkong`(可用区 b / c / d) |
| 集群 | ACK 托管集群 `akernel-hk`,K8s `1.34.10-aliyun.1`,Terway-ENIIP |
| 节点池 | 3 × `ecs.c7.3xlarge`(12 vCPU / 24 GiB),系统盘 100G + 数据盘 100G(ESSD),AliyunLinux3 |
| 节点登录 | SSH key pair `akernel-hk-key` |
| 镜像 | `akerneldev/all-in-one:v0.1.1`(DockerHub 公共,集群内直接拉取) |
| Profile | 本地 `~/AKernel/.akernel/hk/`(ENV=hk) |
| **SDK Server** | `AKERNEL_SERVER_ADDRESS=47.238.229.106`(Traefik 公网 LB,443 TLS + 80 明文双入口) |
| **Grafana** | `http://8.217.239.141:3000`(用户名 `admin`,密码见 `.akernel/hk/grafana-admin-password`) |
| Kubeconfig | `~/AKernel/.akernel/hk/kubeconfig` |
| 节点 cgroup | **v1**(`tmpfs`,满足 v0.1.1 运行时要求) |

---

## 1. 架构与组件

部署由 Terraform 模块 `deploy/terraform/aliyun` 一把拉起,创建以下资源:

- **基础设施**:1×VPC + 3×vSwitch + ACK 集群 + 节点池 + RAM 角色
- **Helm `akernel-core`**(namespace `akernel`):etcd、master、frontend、node DaemonSet、Traefik ingress
- **Helm `akernel-monitor`**(namespace `akernel-monitor`):Prometheus、Grafana、Loki、Tempo

入口模型:Traefik 双入口 —— `websecure:443`(TLS,跑 frontend API + exec websocket)、`web:80`(明文,跑 sandbox 端口转发)。SDK 只需一个 host/IP 即可。

---

## 2. 前置条件

### 2.1 工具

| 工具 | 用途 | 安装方式(本机已装) |
|---|---|---|
| `make` / `terraform` ≥1.5 | 部署编排、infra | 系统自带 |
| `docker` | (本次跳过,用公共镜像) | 已装 |
| `kubectl` / `helm` | 集群验证、helm 操作 | 装到 `~/.local/bin` |
| `aliyun` CLI | 查询云资源(只读) | 装到 `~/.local/bin` |
| `python3` + SDK 依赖 | 跑 SDK / e2e | 见 §6.2 |
| `jq` / `curl` / `git` | 辅助 | 系统自带 |

### 2.2 阿里云凭据

`~/.alicloud_rc` 内容:

```bash
export ALICLOUD_ACCESS_KEY="..."   # RAM AccessKey ID
export ALICLOUD_SECRET_KEY="..."   # RAM AccessKey Secret
export ALICLOUD_REGION="cn-hangzhou"  # ⚠️ 部署时会被覆盖为 cn-hongkong
```

> 该 RAM 账号需要有 VPC / VSwitch / ACK / RAM 角色的创建权限。

### 2.3 ECS SSH key pair(香港区)

香港区默认无 key pair。在控制台「导入」一个本机生成的公钥:

```bash
# 本机生成(ed25519)
ssh-keygen -t ed25519 -f ~/.ssh/aliyun_hk_akernel -C "akernel-hk" -N ""
cat ~/.ssh/aliyun_hk_akernel.pub
```

控制台路径:**ECS → 网络与安全 → 密钥对 → 区域选「中国(香港)」→ 创建密钥对 → 创建类型选「导入密钥对」→ 粘贴公钥 → 确定**。命名(本次为 `akernel-hk-key`)记下,部署时用。

---

## 3. 部署步骤(完整可复现)

> 所有命令在 `~/AKernel` 仓库根目录执行。环境变量 `http_proxy=127.0.0.1:8118` 是翻墙代理,**仅对 GitHub 下载有用,对阿里云 API / PyPI / 公网 LB 必须绕过**(见 §5.7、§6.2、§6.3)。

### 步骤 1:切到 release tag 并初始化子模块

```bash
cd ~/AKernel
git checkout v0.1.1
git submodule update --init --recursive
```

> 本次用公共镜像,**不需要** `make build` / `make push`。子模块初始化仅为让 `make check` 等脚本通过(它们会检查 submodule 源码存在)。

### 步骤 2:安装缺失工具(本机首次)

```bash
mkdir -p ~/.local/bin
# aliyun cli
curl -sL https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz | tar xz -C /tmp
install -m755 /tmp/aliyun ~/.local/bin/aliyun
# kubectl
curl -sLO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -m755 kubectl ~/.local/bin/kubectl
# helm
curl -sL https://get.helm.sh/helm-v3.18.4-linux-amd64.tar.gz | tar xz -C /tmp
install -m755 /tmp/linux-amd64/helm ~/.local/bin/helm
```

### 步骤 3:生成 HK 部署 profile(非交互)

```bash
export PATH="$HOME/.local/bin:$PATH"
make config \
  ENV=hk \
  VENDOR=aliyun \
  NON_INTERACTIVE=1 \
  FORCE=1 \
  REGION=cn-hongkong \
  CLUSTER_NAME=akernel-hk \
  ZONE_IDS="cn-hongkong-b,cn-hongkong-c,cn-hongkong-d" \
  NODE_POOL_KEY_NAME=akernel-hk-key \
  IMAGE_REPOSITORY=akerneldev/all-in-one \
  IMAGE_TAG=v0.1.1 \
  INSTALL_MONITOR=true \
  GRAFANA_PUBLIC_ACCESS=true
```

生成物:`.akernel/hk/{config.env, terraform.tfvars, iam-seed, grafana-admin-password}`。

### 步骤 4:修正配置(踩坑后的必要改动)

`configure.sh` 生成的 tfvars 有几处与香港区 / 当前镜像生态不兼容,**必须手动改 `.akernel/hk/terraform.tfvars`**:

```hcl
# (1) K8s 版本:1.34.3-aliyun.1 港区不可用 → 改为港区可用版本
k8s_version = "1.34.10-aliyun.1"

# (2) etcd 镜像:chart 默认 public.ecr.aws/bitnami/etcd:3.6.8 已失效 → 换可用镜像
etcd_image_repository = "docker.io/bitnamilegacy/etcd"
etcd_image_tag        = "3.5.21"

# (3) traefik busybox sidecar 默认指向未推送的私有 ACR → 关闭
traefik_internal_stats_enabled = false
```

详见 §5 踩坑说明。

### 步骤 5:terraform plan(只读,需人工确认)

```bash
source ~/.alicloud_rc
export ALICLOUD_REGION="cn-hongkong"
make plan ENV=hk
```

> ⚠️ 第一次 plan 时 `terraform init` 要从 GitHub 下载 alicloud provider,而 Make 脚本会 unset 代理导致下载失败(EOF)。解决:先带代理手动 init 一次缓存 provider(见 §5.4),再跑 `make plan`。

### 步骤 6:terraform apply(真正创建资源,产生费用)

```bash
make deploy ENV=hk AUTO_APPROVE=1
```

ACK 集群创建约 8–12 分钟,节点池 + Helm 安装约 3–5 分钟。

### 步骤 7:生成 SDK Token

```bash
make token ENV=hk TTL=24h
# 或长期测试用:make token ENV=hk TTL=100y
```

输出形如 `export AKERNEL_TOKEN='...'`。

---

## 4. 验证部署成功

> 运行 SDK 时**必须绕过代理**(连接的是阿里云公网 IP):命令前加 `env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY`,或先 `unset` 这几个变量。

### 4.1 跑官方 e2e(推荐)

```bash
cd ~/AKernel
export PATH="$HOME/.local/bin:$PATH"
export AKERNEL_SERVER_ADDRESS=47.238.229.106
export AKERNEL_TOKEN="$(make token ENV=hk TTL=24h | sed -n "s/.*AKERNEL_TOKEN='\([^']*\)'.*/\1/p")"
make e2e ENV=hk
```

### 4.2 手动跑 basic_usage.py

```bash
export AKERNEL_SERVER_ADDRESS=47.238.229.106
export AKERNEL_TOKEN="$(make token ENV=hk TTL=24h | sed -n "s/.*AKERNEL_TOKEN='\([^']*\)'.*/\1/p")"
export PYTHONPATH=sdk/python
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  python3 ./sdk/python/examples/basic_usage.py
```

**期望输出:**
```
Sandbox created: <uuid>
Command output: Hello, AKernel!
Remote file: /tmp/hello.txt (18 bytes)
File copy round trip: OK
Background command: pid=... command='sleep 30' running=True
Sandbox state: running
Sandbox terminated.
```

看到以上完整输出即说明:命令执行、文件读写、本地↔远端拷贝、后台进程、沙箱生命周期全部正常 —— 集群在跑真实 gVisor runsc sandbox。

### 4.3 轻量健康检查(不建 sandbox)

```bash
curl -sk https://47.238.229.106/health | head
curl -sk -o /dev/null -w "frontend: %{http_code}\n" https://47.238.229.106/
```

### 4.4 看集群组件状态(运维视角)

```bash
KC=~/AKernel/.akernel/hk/kubeconfig
kubectl --kubeconfig "$KC" get pods -n akernel          # etcd/master/frontend/node/traefik 应 Running
kubectl --kubeconfig "$KC" get pods -n akernel-monitor  # grafana/prometheus/loki/tempo 应 Running
kubectl --kubeconfig "$KC" get nodes                    # 3 节点 Ready
```

判断标准:`akernel-etcd/master/frontend/traefik` 全 `Running`,`node` DaemonSet 至少 1 个 `Running`,节点全 `Ready`。

---

## 5. 部署过程中踩的坑与解决

### 5.1 K8s 版本港区不可用 ❌→✅
- **现象**:`InvalidKubernetesVersion: 1.34.3-aliyun.1 is invalid`
- **原因**:`configure.sh` 硬编码 `k8s_version = "1.34.3-aliyun.1"`,但香港区只支持 `1.36.2 / 1.35.7 / 1.34.10`。
- **解决**:tfvars 改为 `k8s_version = "1.34.10-aliyun.1"`。

### 5.2 etcd 镜像不存在 ❌→✅(最折腾的一个)
- **现象**:`akernel-etcd-0` 一直 `ImagePullBackOff`,`public.ecr.aws/bitnami/etcd:3.6.8: not found`。
- **根因**:chart 默认 etcd 镜像 `public.ecr.aws/bitnami/etcd:3.6.8` 已失效 —— Bitnami 在 2025 年下架了 Docker Hub 上的公共镜像(`docker.io/bitnami/etcd` 全空),ECR 上该 tag 也不存在。
- **试错**:
  - `quay.io/coreos/etcd:v3.5.21` / `gcr.io/etcd-development/etcd:v3.5.21` → 能拉,但是 **distroless,没有 `/bin/sh`**,而 chart 的 etcd 命令是 `/bin/sh -ec "mkdir ...; exec etcd ..."`,直接 `CrashLoopBackOff`。
- **最终方案**:`docker.io/bitnamilegacy/etcd:3.5.21`(Bitnami 的 legacy 仓库,debian 基底、有 shell、`etcd` 二进制、以 `uid=1001` 运行,恰好匹配 chart 的 `dataUser:1001`)。
  ```hcl
  etcd_image_repository = "docker.io/bitnamilegacy/etcd"
  etcd_image_tag        = "3.5.21"
  ```
- **验证方法**:用一次性 pod 在节点上实测镜像是否可拉、是否含 `/bin/sh` + `etcd`:
  ```bash
  kubectl --kubeconfig "$KC" -n akernel apply -f - <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata: {name: imgtest, namespace: akernel}
  spec:
    restartPolicy: Never
    containers:
    - name: c
      image: docker.io/bitnamilegacy/etcd:3.5.21
      command: ["/bin/sh","-c","command -v etcd && echo OK; id; sleep 3"]
  EOF
  kubectl logs -n akernel imgtest
  ```

### 5.3 traefik busybox sidecar 镜像不存在 ❌→✅
- **现象**:`traefik` pod `1/2 ImagePullBackOff`,失败的是 sidecar `registry-vpc.cn-hongkong.aliyuncs.com/akernel/busybox:1.37.0-musl`(pull access denied)。
- **根因**:chart 默认把 traefik 的 internal-stats sidecar 指向「部署用 ACR 私有仓库」里的 busybox,而我们用的是 DockerHub 公共镜像、从未推送 ACR,该仓库不存在。
- **解决**:关闭该 sidecar:
  ```hcl
  traefik_internal_stats_enabled = false
  ```

### 5.4 terraform provider 下载失败 ❌→✅
- **现象**:`Failed to install provider aliyun/alicloud v1.271.0: unexpected EOF`(从 github.com 下载中断)。
- **根因**:本机翻墙代理 `127.0.0.1:8118` 是访问 GitHub 必需的,但 `deploy/scripts/common.sh` 第 9 行会 `unset` 所有 proxy 变量,导致 terraform 直连 GitHub 失败。
- **解决**:绕过 Make,带代理手动 `terraform init` 一次,把 provider 缓存到本地 `.terraform/providers/`,之后 `make plan/deploy` 的 `init` 会复用缓存、不再联网下载:
  ```bash
  cd ~/AKernel/deploy/terraform/aliyun
  source ~/.alicloud_rc; export ALICLOUD_REGION=cn-hongkong
  # 此时 shell 里有 http_proxy,不要 unset
  terraform init
  ```

### 5.5 cgroup 版本风险 ⚠️(已验证 OK)
- **背景**:v0.1.1 的 `deploy/README.md` 明确写「node runtime 要求 cgroup v1,不支持 cgroup v2」,而新版 AliyunLinux3 默认是 cgroup v2。
- **验证**:部署后在节点上检查:
  ```bash
  kubectl --kubeconfig "$KC" -n akernel exec akernel-master-0 -- stat -fc %T /sys/fs/cgroup/
  # 输出 tmpfs = cgroup v1 ✅;输出 cgroup2fs = v2 ❌(需换 cgroup v1 节点镜像)
  ```
- 本次结果:`tmpfs` = **cgroup v1**,符合要求,无需处理。

### 5.6 重复部署时 helm release 残留 ❌→✅
- **现象**:重新 `make deploy` 时 `akernel-core` 报 `cannot re-use a name that is still in use`。
- **根因**:上一次部署被中途 kill,`akernel-core` 处于 `pending-install` 状态(普通 `helm list` 看不到)。
- **解决**:`helm --kubeconfig "$KC" list -A --all` 能看到 pending,`helm uninstall akernel-core -n akernel` 清掉后再 deploy。

### 5.7 pip install 极慢 / 超时 ❌→✅
- **现象**:`pip install openyuanrong-sdk ...` 跑了 10+ 分钟,最终 `ReadTimeoutError: files.pythonhosted.org`。
- **根因**:PyPI 流量被错误地绕到了翻墙代理(带宽有限)。**PyPI 在国内不被墙,不应走代理**。
- **解决**:关代理 + 用阿里云 PyPI 镜像(本机在阿里云,最快):
  ```bash
  env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
    python3 -m pip install -i https://mirrors.aliyun.com/pypi/simple/ \
    'openyuanrong-sdk==0.9.2' 'websockets>=10.0' 'openyuanrong-sandbox'
  ```
  约 1 分钟完成。

### 5.8 第 3 个 node pod 一直 Pending(已知,可接受)
- **现象**:`akernel-node` DaemonSet 有 1 个 pod `Pending`,原因 `Too many pods`(某节点满)+ `NodeAffinity`。
- **根因**:`node_pool_exclusive_eni = true`(独占 ENI 模式)使每节点 pod 配额只有 ~10,系统/addon pod 占满了一个节点。
- **影响**:2/3 节点可用,**够用**;sandbox 容量略降。
- **可选修复**:在 tfvars 设 `node_pool_exclusive_eni = false` 重建节点池,显著提高 pod 密度。

### 5.9 清理过度(教训)
- 排障时曾用一个循环 `helm uninstall` 清理,误删了 ACK 系统 chart(`gateway-api`、`ack-nvidia-device-plugin`)。它们后续由 terraform / ACK 自动恢复,但**清理 helm release 时要按名字精确卸载,不要遍历卸载所有 release**。

---

## 6. 看大盘(Grafana)

### 6.1 登录

| 项 | 值 |
|---|---|
| URL | **http://8.217.239.141:3000** |
| 用户名 | `admin` |
| 密码 | `cat ~/AKernel/.akernel/hk/grafana-admin-password` |

浏览器打开,用上面账号密码登录。AKernel monitor chart 预置了 Prometheus 数据源和 sandbox 相关 Dashboard,登录后在 **Dashboards** 中查看。

### 6.2 命令行确认 Grafana 起来了

```bash
curl -s http://8.217.239.141:3000/api/health
# 期望:{"database":"ok","version":"..."}
```

> Grafana 此入口是 **HTTP 明文(3000 端口)**,密码明文传输。仅测试用;长期使用建议加 TLS 或改为内网访问。

---

## 7. 本地跑 SDK 的环境准备(一次性)

`basic_usage.py` 依赖 `akernel_sdk`,本机按以下方式安装(走阿里云镜像、关代理):

```bash
cd ~/AKernel
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  python3 -m pip install -i https://mirrors.aliyun.com/pypi/simple/ \
  'openyuanrong-sdk==0.9.2' 'websockets>=10.0' 'openyuanrong-sandbox'
```

> 仓库自带 SDK 源码,直接 `export PYTHONPATH=sdk/python` 即可 import,无需 `pip install` SDK 本身(该 SDK 的 PEP 660 editable 安装在本机 setuptools 下会构建出空 `UNKNOWN` 包,故用 PYTHONPATH 方式)。

最小用法:
```python
from akernel_sdk import Sandbox
with Sandbox(cpu=1000, memory=2048) as sb:
    print(sb.commands.run('echo hello').stdout)
```

---

## 8. 常用运维命令

```bash
# 所有命令前先:
cd ~/AKernel
source ~/.alicloud_rc
export ALICLOUD_REGION="cn-hongkong"
KC=~/AKernel/.akernel/hk/kubeconfig

# 重新生成 token
make token ENV=hk TTL=24h

# 打印 SDK 环境
make print-env ENV=hk

# 查看/销毁云资源(销毁会停止计费)
make destroy ENV=hk            # 交互确认
make destroy ENV=hk AUTO_APPROVE=1   # 直接销毁

# 集群运维
kubectl --kubeconfig "$KC" get pods -A
kubectl --kubeconfig "$KC" -n akernel logs <pod>
kubectl --kubeconfig "$KC" describe pod -n akernel <pod>

# 更新部署(改了 tfvars 后)
make plan ENV=hk     # 先看变更
make deploy ENV=hk AUTO_APPROVE=1
```

---

## 9. 注意事项

- **费用**:3×`ecs.c7.3xlarge` + 2 个 SLB(Traefik、Grafana)+ ESSD 磁盘持续计费。不用时 `make destroy ENV=hk`。
- **Token 安全**:`make token` / `make print-env` 输出的是 JWT 密钥,不要提交到 git 或贴到公开场合。泄露后只能轮换 IAM seed(`.akernel/hk/iam-seed`)使旧 token 失效。
- **不要提交本地状态**:`.akernel/`、terraform.tfstate、kubeconfig、token、iam-seed 都在 `.gitignore` 内,切勿提交。
- **代理边界**:翻墙代理(8118)只用于 GitHub/Google 类被墙站点;阿里云 API、PyPI、公网 LB、Pod IP、集群内网地址都应**绕过代理**。

---

*文档生成于部署验证通过后。如需复现,按 §3 顺序执行,并务必应用 §4(即步骤 4)的 tfvars 修正。*
