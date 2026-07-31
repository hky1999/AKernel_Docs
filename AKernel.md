# 从 Computer Use到 Datacenter Use：如何让 AI Agent 像调用函数一样驱动数据中心？

link: https://mp.weixin.qq.com/s/0Qce15eCqoTpXUk26pQsQQ

AKernel 是一个面向 Agent 的云操作系统，我现在要跟这个系统的作者合作写一篇 OSDI/SOSP 级别的顶会论文，主要想要给大家展示一下在 Agent 的时代，面向 Agent 的云端容器框架（或者云操作系统，就跟 k8s 一样）应该长什么样。

整个论文的行文思路想要按照 [SigmaOS](./References/Szekely%20et%20al_2024_Unifying%20serverless%20and%20microservice%20workloads%20with%20SigmaOS.pdf) 、[YuanRong](./References/Chen%20等%20-%202024%20-%20YuanRong%20A%20Production%20General-purpose%20Serverless%20System%20for%20Distributed%20Applications%20in%20the%20Cloud.pdf) 还有 [AFaaS](./References/Chai%20et%20al_2025_Fork%20in%20the%20Road.pdf)来。


## References

* [从 Computer Use到 Datacenter Use：如何让 AI Agent 像调用函数一样驱动数据中心？](https://mp.weixin.qq.com/s/0Qce15eCqoTpXUk26pQsQQ)
* [重磅！华为开源业界首个 Serverless 分布式计算引擎 openYuanrong，单机体验编程、极致分布式运行性能](https://www.infoq.cn/article/e7qqfyya9gebhrsc7iyt)
* [YuanRong](./References/Chen%20等%20-%202024%20-%20YuanRong%20A%20Production%20General-purpose%20Serverless%20System%20for%20Distributed%20Applications%20in%20the%20Cloud.pdf)
* [SigmaOS](./References/Szekely%20et%20al_2024_Unifying%20serverless%20and%20microservice%20workloads%20with%20SigmaOS.pdf)
* [AFaaS](./References/Chai%20et%20al_2025_Fork%20in%20the%20Road.pdf)
* [E2B: AI Sandboxes](https://e2b.dev/)
* [阿里开源的安全沙箱解决方案 OpenSandbox](https://mp.weixin.qq.com/s/5gQbCKjxFv-FBGp_2qLtIA)
* [腾讯开源Cube Sandbox：60毫秒冷启动的AI沙盒运行时](https://mp.weixin.qq.com/s/B3jaoCcYsSXt0epopijZPA)
* [Kubernetes won the container decade. Google’s Agent Substrate wants the next one.](https://thenewstack.io/kubernetes-ai-agent-runtime/)
* [Kubernetes 统治了容器时代，谷歌 Agent Substrate 意在拿下下一个十年](https://mp.weixin.qq.com/s/5YtXEhywk_JIJ04dknY0uQ)
