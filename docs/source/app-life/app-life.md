# agentgateway 应用生命周期


## 概述

本文分析 AI Agent 总线网关 [agentgateway](https://github.com/agentgateway/agentgateway) 源码，尝试解释服务生命周期管理相关的组件和协作关系。从服务各组件的启动及初始化，端口监听，到服务终止信号如何在各组件之间传播，以及服务优雅关闭(Drain)相关的实现。

由于我对 Rust ，特别是 tokio 的异步程序风格了解有限，如其中有错漏，还请指出。



AI Agent 总线网关 agentgateway 实现分析 系列：

1. [AI Agent 总线网关 agentgateway 实现分析 Part 1](https://blog.mygraphql.com/zh/posts/ai/ai-devops/agent-gateway/agentgateway-impl/)
2. AI Agent 总线网关 agentgateway 实现分析 Part 2（本文）

建议顺序阅读。



## 引

如果让你开发一个 Reverse Http Proxy。你觉得最困难、最复杂的设计是什么？是线程模型？协议解码？流控Buffer ?  我觉得是如何让服务优雅关闭(Drain)，甚至不断连接热升级。

和应用大部分代码其实是花在异常处理上一个道理，Proxy 的很多代码其实是花在优雅关闭(Drain)上。而明白服务的：

- 核心组件或子服务有什么
- 组件或子服务的初始化过程
- 组件或子服务的优雅关闭(Drain)

这几点是理解服务架构的重点，即服务的生命周期。下面，我们就这几方面进行分析。



## 服务的生命周期



agentgateway 有几大子服务：

- admin service
- metrics server
- readiness service
- **Gateway service** 

其中，最后两个运行于 worker 线程池中，其它在主线程上。真正做核心负载的，当然是 Gateway Service 了。



在停止服务时，要，当然要通知到这些子服务了。agentgateway 大量使用了 Rust 的 channel 机制实现各异步 future 之间的通讯。



1. `app.rs > Bound::wait_termination(self)` 是服务停止入口，也负责通知到各子服务。
2. 它接收操作系统 `SIGTERM` 信号后
3. 触发优雅关闭(Drain)流程：`start_drain_and_wait()` 发送消息到 `Signal(DrainTrigger)`。

4. 各子服务监听 `Watch(DrainWatcher)` ，在收到 `Signal(DrainTrigger)` 发出的消息后，执行 Drain 操作：如 HTTP 连接关闭通知等等。
5. 在 Drain 操作执行完成后，会 feedback 到  `Signal(DrainTrigger)`
6. 最终所有子服务都 feedback 后。整体服务可以停止了。



以下就是这个流程的详述：



:::{figure-md}
:class: full-width

<img src="agentgateway-channels.drawio.svg" alt="图：agentgateway 应用生命周期协作">

*图：agentgateway 应用生命周期协作*  
:::
*[用 Draw.io 打开](https://app.diagrams.net/?ui=sketch#Uhttps%3A%2F%2Fagentgateway-insider.mygraphql.com%2Fzh_CN%2Flatest%2F_images%2Fagentgateway-channels.drawio.svg)*


## 结语

我深度研究过 Envoy Proxy 的 C++ 代码。可以说，它重度使用了 基于 OOP 和多态的设计方法，基于事件驱动 Callback 的组件子系统解藕方法。这让代码看起来比较沉长，概念术语词汇繁多，有时已经有点 Java 的感觉了。

而 agentgateway 或者是其参考设计的项目 [Istio ztunnel](https://github.com/istio/ztunnel) （这两个项目有共同的公司 Solo.io 以及共同的开发者 John Howard） 使用了 Tokio + Rust async，则有点像 Golang 的 goroutines 。Reddit 上有一个比较：[How Tokio works vs go-routines?](https://www.reddit.com/r/rust/comments/12c2mfx/how_tokio_works_vs_goroutines/) 。

单从代码阅读上，风格简单，实用主义，不过渡做作的  Tokio + Rust async 似乎更容易上手。当然前提是了解基本的 Rust async + Tokio 。



### 为什么要研究 agentgateway

这是个好问题。我认为 AI 应用已经到来， AI  应用的基础设施也需要完善。而作为传统的基础设施程序员（非应用程序员），与其等待 AI 替代自己的工作，不如先让 AI  依赖我的工作，然后希望大家可以和平共存。而 gateway 类型的基础设施在大企业的 AI  治理内，注定是不可缺少的。


