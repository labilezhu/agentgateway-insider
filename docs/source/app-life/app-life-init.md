# agentgateway 服务初始化流程



## 初始化流程

下面说说初始化流程和线程模型。主要的线程有三类：

- main thread，线程名 agentgateway
- main spawn thread，线程名也是 agentgateway
- agentgateway workers，线程名格式： agentgateway-N 


:::{figure-md}
:class: full-width

<img src="index.assets/agentgateway.drawio.svg" alt="图：Agentgateway 初始化流程">

*图：Agentgateway 初始化流程*  
:::
*[用 Draw.io 打开](https://app.diagrams.net/?ui=sketch#Uhttps%3A%2F%2Fagentgateway-insider.mygraphql.com%2Fzh_CN%2Flatest%2F_images%2Fagentgateway.drawio.svg)*


浏览这种 Rust + tokio 异步编程风格的代码，对于 rust 新手，是有点废脑。好在我之前学过 golang 的 Goroutines。大概看到门路。每线程可以在线程上下文中绑定一个 [tokio::runtime::Runtime](https://docs.rs/tokio/1.47.1/tokio/runtime/index.html) （相当于一个带线程池的 scheduler）。只要关注所有 “spawn” 相关的代码。
