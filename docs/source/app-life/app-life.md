# agentgateway 应用生命周期


## 概述

本文分析 AI Agent 总线网关 [agentgateway](https://github.com/agentgateway/agentgateway) 源码，尝试解释服务生命周期管理相关的组件和协作关系。从服务各组件的启动及初始化，端口监听，到服务终止信号如何在各组件之间传播，以及服务优雅关闭(Drain)相关的实现。

由于我对 Rust ，特别是 tokio 的异步程序风格了解有限，如其中有错漏，还请指出。



## 服务的生命周期



agentgateway 有几大子服务：

- admin service
- metrics server
- readiness service
- Gateway service

其中，最后两个运行于 worker 线程池中，其它在主线程上。真正做核心负载的，当然是 Gateway Service 了。



```{toctree}
app-life-init.md
app-life-shutdown.md
```