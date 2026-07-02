从 Computer Use到 Datacenter Use：如何让 AI Agent 像调用函数一样驱动数据中心？

尽管云原生基础设施的工业化程度已经相当成熟，但要让 Agent 真正按需调用驱动中心的算力资源，二者之间仍存在难以忽视的鸿沟。

C1: 基础设施使用门槛高
    how to build your own cluster
    基础设施栈——Kubernetes、节点侧操作系统内核、上层调度系统、分布式存储

C2: 数据与算力的分布式调度
    Agent 真正需要的是"算力在哪里，就在哪里搭一套足够可用的存储系统"。

C3: 高并发下的实例冷启动问题 （水平 scaling 问题）
    万级并发下，镜像拉取如何解决？ 

C4: Agent 的运行环境 （垂直 scaling 问题）
    Agent 需要在一个安全的沙箱中运行，避免对宿主机造成破坏。
    Agent 不再是一个无状态的短任务容器，而是一个需要拥有状态保存、状态迁移和状态续跑能力的工作单元.
    什么样的沙箱符合 Agent 的 workload 需求？


解决 C1:
    提出我们的 云操作系统 AKernel

解决 C2:
    openYuanrong 分布式调度与数据系统

解决 C3:
    1. 基于 Rust FUSE 实现的 distill-fs，负责镜像的按需懒加载； 节点上的镜像组件不会在沙箱启动时预先下载完整镜像，而是在用户代码真正访问到镜像内具体文件时，才按需触发对应内容的拉取。大量从未被访问的文件层完全无需下载，从而大幅降低启动阶段的 I/O 开销。
    2. Dragonfly P2P 加速组件，保障镜像的分发效率。在集群内建立镜像缓存层，通过 P2P 网络在节点间分发镜像内容，将大部分下载流量消化在集群内部，避免所有节点同时向外部仓库发起请求。
    3. AFaaS 按需沙箱技术。基于 gVisor 演进的 nanovisor，提供轻量级且高安全隔离的沙箱运行时；

解决 C4:
    多种沙箱运行时
    AKernel 已原生支持 Jupyter 与 PyTorch 等轻量级且具备强安全隔离能力的沙箱运行时
    SKernel + AFaaS,需要强 Linux 兼容则回退到 Firecracker
    除 Sandbox 外，AKernel 亦全面覆盖了 FaaS 与 Spark 等多元化 Workload 的调度与执行。