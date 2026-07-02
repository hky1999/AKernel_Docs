# AKernel 论文大纲

这个项目是为了撰写一篇 面向 Agent 的云操作系统，AKernel 的顶会论文（OSDI/SOSP 级别）而创建的。
具体内容与期望参考 [AKernel.md](./AKernel.md) 中的内容。

* [outline_draft.md](./outline_draft.md) 是论文大纲的初稿，包含了论文的核心论点、Motivation、设计原则和解决方案。
* [Paper_outline.md](./Paper_outline.md) 是论文大纲的最新版本，还没调研完成，没有最终定好这个论文要咋写。


## 开发规则

Survey 文档都放在 `./Surveys` 目录下，带上详细（精确到秒）的时间戳，方便追溯，比如 `./Surveys/${时间戳}-${主题}.md`。

Survey 文档的内容要求包括详细数据，还有详细的参考文献、来源链接，方便后续引用与查看。

在进行 Survey 的时候**务必**启用多个 subagent 进行并行搜索。

Survey 范围涵盖近几年的 SOSP/OSDI/NSDI/USENIX ATC/EuroSys/FAST/ASPLOS 等顶会论文，尤其是与云原生、分布式系统、操作系统、容器、虚拟化、FaaS、Serverless、微服务、Agent 相关的论文。
当然，agent 是 AI 相关，也可以参考近几年 NeurIPS/ICLR/ICML/AAAI/CVPR 等顶会论文，尤其是与 LLM、RLHF、Agent、Multi-Agent、AutoGPT、LangChain、AgentOS、AgentBench 等相关的论文。
