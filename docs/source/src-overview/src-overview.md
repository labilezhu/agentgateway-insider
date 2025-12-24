# agentgateway 项目源码概览


![图：agentgateway 代理 Agent 的对外连接，包括 MCP 服务器、AI Agent 和 OpenAPI](./index.assets/architecture.svg)

*图：agentgateway 代理 Agent 的对外连接，包括 MCP 服务器、AI Agent 和 OpenAPI*

*(source: https://agentgateway.dev/docs/about/architecture/)*



## 概述

本文通过分析 [agentgateway](https://github.com/agentgateway/agentgateway) 的源码，初步了解主要初始化过程。希望对想深入学习 agentgateway 实现的读者提供一些大方向上的参考和指引。

由于我对 Rust ，特别是 tokio 的异步程序风格的了解有限，如果其中有错漏，还请指出。



## 引

[Critical thinking](https://www.criticalthinking.org/pages/defining-critical-thinking/766) 的人，首先得回答一个问题，为什么我们需要另一个 proxy ? Nginx/HAProxy/Envoy Gateway .... 不足够吗？



之所以需要一个新的 gateway，而不是直接复用现有的 API Gateway、Service Mesh 或传统反向代理，是因为 **AI agent 系统有独特的需求**，现有网关无法很好满足：

1. **协议差异**
    AI agent 之间的交互常用 MCP、A2A 等新协议，这些涉及长连接、双向通信、异步消息流，并非传统网关主要支持的 HTTP/REST 或 gRPC 模式。
2. **状态与会话管理**
    传统网关多为无状态转发，而 agent 交互通常需要维护长时上下文、身份状态和会话信息。
3. **安全与多租户**
    Agent 生态需要更精细的身份验证、权限控制和隔离能力，以保证不同用户、不同 agent 之间安全可靠地共享工具。
4. **可观测性与治理**
    AI agent 的调用链和事件流复杂，需要更强的监控、指标收集、追踪和流量治理能力，而传统网关在这方面偏弱。
5. **工具与服务虚拟化**
    Agentgateway 能将分散的 MCP 工具或外部 API 统一抽象、注册和暴露，方便 agent 动态发现和调用。

**现有网关在协议支持、状态管理、安全性和可观测性上都不适配 AI agent 场景，因此需要一个为 agent 专门设计的新一代 gateway。**



下面对比表，把 **传统 API Gateway 和 Agentgateway** 在几个关键能力上的差异列出来：

| 能力/特性           | 传统 API Gateway          | Agentgateway                                    |
| ------------------- | ------------------------- | ----------------------------------------------- |
| **协议支持**        | HTTP/REST、gRPC 为主      | 专为 MCP、A2A 等 agent 协议设计，支持双向、异步 |
| **连接模式**        | 短连接，请求-响应         | 长连接、双向流、状态化会话                      |
| **状态管理**        | 无状态转发                | 维护 agent 会话、上下文与身份                   |
| **安全控制**        | 认证、鉴权（偏 API 用户） | 精细化权限控制，多租户隔离，支持工具级别授权    |
| **可观测性**        | 基础日志/指标             | 针对 agent 的调用链跟踪、事件监控、指标收集     |
| **流量治理**        | 限流、熔断、路由          | 结合 agent 行为的智能治理（会话级/工具级）      |
| **工具虚拟化/发现** | 无                        | 支持 MCP 工具/服务注册、抽象与动态发现          |
| **典型场景**        | 传统 API 管理，面向客户端 | AI agent 平台，工具/LLM/agent 互联              |

可以看到，Agentgateway 的定位不是替代传统 API Gateway 或 Service Mesh，而是 **补齐 AI agent 场景下的协议适配、会话管理、安全和治理空白**。



### Agentgateway 介绍

Agentgateway 是一个开源且跨平台的数据平面，专为 AI agent 系统设计，能够在 agent、MCP 工具服务器与 LLM 提供者之间建立安全、可扩展、可维护的双向连接。它弥补了传统网关在处理 MCP/A2A 协议中存在的状态管理、长会话、异步消息、安全、可观测性、多租户等方面的不足，提供统一接入、协议升级、工具虚拟化、身份验证与权限控制、流量治理、指标与追踪等企业级能力，还支持 Kubernetes Gateway API、动态配置更新以及内嵌开发者自服务门户，帮助快速构建和扩展 agent 化 AI 环境。



上面是 AI 生成的介绍，我还是说句人话吧 :)  我认为现阶段的 agentgateway 更像一个 AI Agent 应用的 outbound bus(外部依赖总线) 。



## 代码结构

应该算典型的 Rust 项目吧，我的 Rust 是刚学回来的，不是也不要打我。`Cargo.toml` 中列出了所有依赖。也有几个 *.md 文档，`README.md` 套路我就不说了，`DEVELOPMENT.md` 是比较好的开发操作入门。



当我看到 `CODE_OF_CONDUCT.md` 还是有点惊讶的：

```md
Usage of AI during contributions must abide by the [Linux Foundation Generative AI policy](https://www.linuxfoundation.org/legal/generative-ai).
In addition, please:
* Refrain from generating issues, comments, or PR descriptions with AI.
* Refrain from "vibe coding". AI should be used to assist and accelerate code that you (the human!) would write on your own.
* When using (non-trivial) AI assistance, please indicate this.

These policies ensure the code base is kept at a high level of quality.
Additionally, it ensures maintainers do not waste time reviewing low-quality AI-generated code.
---
在贡献过程中使用 AI 必须遵守 [Linux 基金会生成式 AI 政策](https://www.linuxfoundation.org/legal/generative-ai)。
此外，请注意：

* 禁止使用 AI 生成问题（issue）、评论或 PR 描述。
* 禁止进行所谓的 “vibe coding”。AI 应仅用于辅助和加速你（人类开发者）本应自己编写的代码。
* 当使用了（非简单的）AI 辅助时，请明确标注。

这些政策确保代码库始终保持高质量水准。
同时也避免维护者浪费时间去审查低质量的 AI 生成代码。
```

一个 AI 向项目在部分关键任务中，禁止使用 AI ，算不算是个有趣的事？



这是一个典型的基于 **Cargo Workspace** 的 Rust 项目，意味着它由多个相互关联的独立包（称为 `crates`）组成。以下是主要模块和关键位置的说明：

### 顶层结构

- `Cargo.toml`: 这是 **Workspace 的根配置文件**。它定义了整个工作区包含哪些 `crates`，并可能定义所有成员共享的依赖项和配置。
- `crates/`: **这是项目最核心的源码目录**。所有的 Rust 逻辑都存放在这里，每个子目录都是一个独立的 `crate`。
- `ui/`: 这是一个 **前端项目**，从 `next.config.ts` 和 `package.json` 文件来看，它是一个使用 **Next.js (React)** 构建的 Web 用户界面。它与后端的 Rust 代码是分离的。
- `examples/`: 包含各种**用法示例**。每个子目录（如 `basic`, `tls`, `openapi`）都演示了 `agentgateway` 的一种特定功能或配置方式，这对于理解如何使用此项目非常有帮助。
- `schema/`: 存放**配置文件的数据结构定义 (Schema)**。`local.json` 和 `cel.json` 定义了配置文件的格式和验证规则。
- `go/`: 包含一些 Go 语言的源码。从文件名 `*.pb.go` 来看，这些是由 Protocol Buffers (`proto`) 定义自动生成的代码，用于跨语言（Rust/Go）的服务间通信。
- `architecture/`: 包含项目**架构设计的文档**。

### `crates/` 目录下的重要模块 (Crates)

这里是 Rust 代码的核心，每个 `crate` 都有明确的职责：

- `agentgateway`: **核心代理逻辑 Crate**。
  - **位置**: `crates/agentgateway/`
  - **说明**: 这是项目的主要库，实现了代理（Proxy）、路由、中间件等所有核心功能。
- `agentgateway-app`: **应用程序入口 Crate**。
  - **位置**: `crates/agentgateway-app/`
  - **说明**: 这是一个二进制（binary）`crate`。它的主要作用是引用 `agentgateway` 这个库 `crate`，并构建成最终的可执行文件。程序的 `main` 函数应该就在这里。这是 Rust 项目中常见的“库 + 应用”分离模式。
- `core`: **核心类型和工具 Crate**。
  - **位置**: `crates/core/`
  - **说明**: 通常用于存放被工作区中多个其他 `crates` 共享的基础数据结构、traits、常量和工具函数。
- `xds`: **xDS API 实现 Crate**。
  - **位置**: `crates/xds/`
  - **说明**: "xDS" 是云原生领域（尤其是在 Envoy 代理中）广泛使用的一组发现服务 API。这个 `crate` 很可能实现了 xDS 客户端，用于从控制平面动态获取配置（如路由、服务发现等）。`proto` 子目录和 `build.rs` 文件印证了它需要编译 Protobuf 文件。
- `hbone`: **HBONE 协议 Crate**。
  - **位置**: `crates/hbone/`
  - **说明**: HBONE (HTTP-Based Overlay Network Encapsulation) 是 Istio 中用于零信任网络的一种隧道协议。这个 `crate` 专门负责处理 HBONE 连接和通信。
- `a2a-sdk`: **Agent-to-Agent SDK Crate**。
  - **位置**: `crates/a2a-sdk/`
  - **说明**: Agent-to-Agent SDK
- `mock-server`: **测试用的模拟服务器**。
  - **位置**: `crates/mock-server/`
  - **说明**: 用于在集成测试或端到端测试中模拟后端的 HTTP 服务。
- `xtask`: **构建和任务自动化 Crate**。
  - **位置**: `crates/xtask/`
  - **说明**: 这是一种使用 Rust 编写项目自动化脚本（如构建、测试、部署）的流行模式，作为 `Makefile` 或 shell 脚本的替代方案。

### 总结

这个项目是一个功能丰富的**服务网关**，其架构清晰地分离了不同的关注点：

1. **核心业务逻辑**在 `crates/agentgateway`。
2. **程序入口**在 `crates/agentgateway-app`。
3. **底层网络协议**（如 HBONE, xDS）被抽象到独立的 `crates` (`hbone`, `xds`) 中。
4. **用户界面** (`ui/`) 和**后端** (`crates/`) 是完全解耦的。
5. **示例** (`examples/`) 提供了丰富的使用场景，是学习和使用的好起点。



## 开发环境

我用 vscode container 。因为其中有 `.devcontainer.json` ，在 container 中运行开发环境的好处是，每个开发环境一配置都是一致，减少了很多依赖问题、工具链问题、版本问题、沟通成本。

阅读源码，特别是 Rust 特别是 tokio 异步程序风格的源码。我需要在 vscode 安装一个 AI 先生：Gemini Code Assist


### Build agentgateway

agentgateway 自带一个 web 管理界面项目： `agentgateway/ui` 。它需要单独构建。构建需要安装 `npm` 。 具体说明见：`ui/README.md` 。 如果你用的是 vscode container ：
```bash
sudo apt update
sudo apt install npm

npm install
```

### Debug agentgateway

vscode 会生成 launch.json :

```json
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug executable 'agentgateway'",
            "cargo": {
                "args": [
                    "build",
                    "--bin=agentgateway",
                    "--package=agentgateway-app",
                    "--features",
                    "agentgateway/ui"
                ],
                "filter": {
                    "name": "agentgateway",
                    "kind": "bin"
                }
            },
            "args": ["--file", "/workspaces/agentgateway/examples/basic/labile-config.yaml"],
            "cwd": "${workspaceFolder}",
        },
```



需要注意的是，我在 cargo 配置中加入了 `--features` 。 这相当于 C++ 项目的 MACRO 配置。要打开 `agentgateway/ui` 才有 ui 功能，以下 `crates/agentgateway/src/app.rs` 代码才生效：

```rust
	#[cfg(feature = "ui")]
	admin_server.set_admin_handler(Arc::new(crate::ui::UiHandler::new(config.clone())));
	#[cfg(feature = "ui")]
	info!("serving UI at http://{}/ui", config.admin_addr);
```





## 实现分解

如果以上内容被你怀疑是 AI 生成的，那么以下就是人类手写的本文核心内容。



### 初始化流程

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





## AI 读代码大法

这种 Rust + tokio 异步编程风格的代码对于新手，不太友好。所以我在 vscode 安装一个 AI 助手：Gemini Code Assist。这使我不需要先阅读一本厚厚的 Rust 书，才开始阅读项目代码。这样直接读开源代码去学习编程语言的方法，也比之前一步步踏实学习语言有趣得多（虽然有时感觉不踏实）。如果你觉得 Vibe Coding 有点荒谬，那么 Vibe Code Reading 就相对靠谱得多。以下展示一些实例：



和搜索引擎的区别是，他直接在工作的 context 信息下，回答问题。而不需要人为提炼出信息出来。或者，在 AI 普及年代，会提问题变得更重要。



## 参考

- [tokio tutorial](https://tokio.rs/tokio/tutorial/spawning)

