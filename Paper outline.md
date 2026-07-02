# AKernel 论文大纲

## 暂定题目

**AKernel: 面向 Datacenter Use 的 Agent-Native 云操作系统**

## 一句话主张

AI Agent 正在从“使用一台虚拟电脑”走向“驱动整个数据中心”。
AKernel 要证明：Agent 时代需要一种新的云操作系统，把执行、状态、数据局部性、安全隔离和集群构建统一成可编程的系统抽象。

## 核心论点

AI sandbox API 正在商品化。真正的系统问题不是如何启动一个 sandbox，而是如何让成千上万个有状态、突发式、产物密集、受策略约束的 Agent 执行单元，像云端一等进程一样被调度、恢复、迁移和管理。

AKernel 的核心抽象是：**Agent Process** 运行在 **Agent Realm** 中，读写 **Agent Object**，并通过 **Datacenter Function** 被 Agent 像调用函数一样驱动。

## Motivation

### M1. Computer Use 太小了

Computer Use 给 Agent 一台机器。Datacenter Use 给 Agent 弹性算力、分布式数据、高并发执行和持久状态。

### M2. Sandbox 必要，但已经不够

E2B、Modal、Daytona、OpenSandbox、Cube Sandbox、Kubernetes Agent Sandbox 已经提供 sandbox 生命周期、命令执行、文件 API、网络、快照等能力。

因此，AKernel 不能把“提供一个 sandbox SDK”作为创新点。真正缺失的是 Agent 执行的 OS-level 语义。

### M3. Agent Workload 既不是 FaaS，也不是 Microservice

Agent workload 的特征是：

- 多步骤、有状态
- 并发探索和评测时高度突发
- 产生大量文件、日志、代码仓库、checkpoint、数据集等中间产物
- 经常等待 LLM、工具、人审或外部 API
- 运行不可信代码，并需要受控访问网络、凭证和数据

### M4. Datacenter Use 需要跨层优化

一次 Agent 任务可能包含调度、sandbox 启动、镜像懒加载、依赖导入、对象读取、网络策略配置、执行、checkpoint、restore 和产物提交。

只优化 cold start 不够。

## 核心抽象

### Agent Process

AKernel 中最小的可调度 Agent 执行单元：一个有状态、强隔离、携带策略的云端进程。

### Agent Realm

一个完整 Agent 任务、多 Agent workflow、RL rollout 或用户会话的隔离域和资源域。

### Agent Object

对文件、checkpoint、日志、数据集、模型产物、package cache、浏览器 trace 和中间状态的统一表示。

### Datacenter Function

面向 Agent 的调用语义：表面上像函数调用，实际展开为分布式执行、对象放置、sandbox 生命周期和状态管理。

## 设计原则

1. **Useful start，而不只是 cold start**：衡量到达第一次有意义动作的时间。
2. **状态是一等公民**：checkpoint、restore、fork、migrate、commit 是生命周期原语。
3. **计算跟着状态走**：调度要感知 image cache、checkpoint location 和 object locality。
4. **策略跟着执行走**：网络、凭证、数据访问和审计策略绑定到 Agent Process。
5. **Goodput 优先于 throughput**：优化 SLO 内单位成本完成的正确任务数。
6. **Build Your Own Cluster**：Agent 应该能构建和操作自己的执行集群。

## 系统架构

### Control Plane

SDK、CLI、Gateway、scheduler、Agent Realm manager。

### Execution Plane

Sandbox Daemon、AFaaS、nanovisor、runtime selection、process lifecycle。

### Data Plane

openYuanRong Data System、Agent Object、KV/Object storage、shared memory、cross-node transfer。

### Image Plane

distill-fs lazy loading、Dragonfly P2P distribution、image cache locality。

### State Plane

Checkpoint/Restore、hibernation、restore locality、failure recovery。

### Policy and Network Plane

eBPF NAT、双向代理、capability enforcement、audit/provenance。

### Deployment Plane

基于 Terraform + Helm 的多云 Build Your Own Cluster。

## 主要贡献

### C1. Agent Workload Characterization

基于真实 trace 证明 Agent workload 是有状态、突发式、产物密集、易空闲且受策略约束的。

### C2. Agent-Native Cloud OS Abstractions

提出 Agent Process、Agent Realm、Agent Object 和 Datacenter Function，作为 Datacenter Use 的核心 OS 抽象。

### C3. AKernel 跨层系统实现

基于 openYuanRong、AFaaS、镜像懒加载、P2P 分发、Checkpoint/Restore 和策略感知网络，实现上述抽象。

### C4. 端到端评估

在真实 Agent workload 上评估 task goodput、cost-to-solution、time-to-useful-action、资源节省和 locality 收益。

## 论文结构

### 1. Introduction

从 Computer Use 到 Datacenter Use。  
为什么 sandbox API 不够。  
为什么 Agent workload 需要云操作系统。

### 2. Background and Related Systems

Cloud OS：SigmaOS、YuanRong。  
Serverless 与状态：Cloudburst、Beldi、Pocket、SONIC。  
冷启动与隔离：AFaaS、Firecracker、SEUSS、FaaSnap。  
AI sandbox 平台：E2B、Modal、Daytona、OpenSandbox、Cube Sandbox。

### 3. Agent Workload Characterization

Trace 方法。  
Workload 特征。  
系统设计启示。

### 4. AKernel Abstractions

Agent Process。  
Agent Realm。  
Agent Object。  
Datacenter Function。  
生命周期语义。

### 5. System Design

Control plane。  
Execution plane。  
Data plane。  
Image plane。  
State plane。  
Policy/network plane。  
Deployment plane。

### 6. Implementation

AKernel 如何整合 openYuanRong、AFaaS、Data System、distill-fs、Dragonfly、eBPF NAT、Terraform 和 Helm。

### 7. Evaluation

Workload trace 结果。  
Time-to-useful-action。  
Checkpoint/Restore 效率。  
Object locality。  
Burst scaling。  
Policy overhead。  
Build Your Own Cluster。

### 8. Production Experience

内部部署经验。  
Agentic RL、FaaS、Spark 和 Sandbox workload。  
小团队运维。  
AI-assisted debugging。

### 9. Discussion

GPU/NPU 支持。  
Windows/macOS/Android sandbox。  
Replay 限制。  
安全边界。  
与闭源 sandbox 平台的对比限制。

## Evaluation Plan

### Workloads

- Coding agent tasks
- Agentic RL rollout and evaluation
- Data analysis tasks
- Terminal/ops tasks
- FaaS and Spark mixed workloads

### Baselines

- Kubernetes + containerd/gVisor
- openYuanRong baseline
- E2B-like single-sandbox baseline
- AKernel without checkpoint/restore
- AKernel without locality-aware scheduling
- AKernel without lazy image/P2P distribution

### Metrics

- time-to-useful-action
- successful tasks per hour
- cost per successful task
- resource-hours saved
- checkpoint and restore latency
- object cache hit rate
- cross-node traffic
- p50/p95/p99 task latency
- policy enforcement overhead

## 最小证据包

1. 真实 Agent workload trace。
2. 与 AI sandbox 平台的能力对比。
3. 端到端 task-goodput 实验。
4. Checkpoint/Restore 资源节省实验。
5. Object locality 实验。
6. Burst-scale startup/restore 实验。
7. Build Your Own Cluster 部署实验。

## 最终定位

AKernel 不是另一个 AI sandbox。它是一个面向 Agent 的云操作系统，把执行、状态、数据、策略和部署统一成 Datacenter Use 的可编程底座。
