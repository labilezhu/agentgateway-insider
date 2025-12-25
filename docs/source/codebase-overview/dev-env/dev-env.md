# 开发环境安装与配置

本书推荐使用 Visual Studio Code 作为主要的代码阅读和调试工具。推荐用 vscode container 。因为项目中已经有 `.devcontainer.json` ，在 container 中运行开发环境的好处是，每个开发环境一配置都是一致，减少了很多依赖问题、工具链问题、版本问题、沟通成本。

(diagram-vscode-code-link-setup)=
## 源码导航图链接到 vscode 源码
正如我在 {ref}`/ch0/index.md:互动图书` 一节说的。书中的图带 ⚓ 图标，双击可链接到本地 vscode 的源码位置。读者拥有源码导航图与源码无缝切换的阅读体验。

因为 Draw.io vscode 插件 的限制，drawio 图文件必须放在 vscode workspace 里一个与本书开发环境相同的路径下，才能正确打开链接到本地源码的位置。我已经把书中的 drawio 图与一些资源文件，放在 https://github.com/labilezhu/pub-diy.git 仓库中。


1. 需要安装 [Draw.io vscode 插件](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio)

2. 首先 clone `pub-diy` 仓库：

```bash
cd ~/
git clone https://github.com/labilezhu/pub-diy.git
```

可见本书的所有图与资源文件均在 `pub-diy` 仓库的 `ai/agentgateway/ag-dev` 目录下。


3. 把 pub-diy 仓库挂载到开发容器

然后修改一下 agentgateway 源码库的 `.devcontainer.json` 文件：

```json
{
    "image": "mcr.microsoft.com/devcontainers/rust:latest",
    "mounts": [
        "source=/home/your_home/pub-diy,target=/workspaces/agentgateway/diy-log,type=bind,consistency=cached"
    ]
}
```
即把 pub-diy 仓库挂载到容器的 `/workspaces/agentgateway/diy-log` 目录下。然后在 vscode 下执行 `Rebuild Container` 重建开发容器。完成后，所有图与资源文件均在容器里的 `/workspaces/agentgateway/diy-log/ai/agentgateway/ag-dev` 目录下。


## 构建 agentgateway

agentgateway 自带一个 web 管理界面项目： `agentgateway/ui` 。它需要单独构建。构建需要安装 `npm` 。 具体说明见：`ui/README.md` 。 如果你用的是 vscode container ：
```bash
sudo apt update
sudo apt install npm

npm install
```

## Debug agentgateway

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
