# 安装 AgentFS

稳定版 [v0.3.3](https://github.com/meglinge/agentfs-releases/releases/tag/v0.3.3) 提供不绑定仓库的 MCP 入口，同一连接可以管理多个仓库和工作区，并自动同步本地 Git 提交。原生工具负责文件编辑，MCP 保留工作区、版本协作和快速查询。二进制和安装脚本公开分发在 [meglinge/agentfs-releases](https://github.com/meglinge/agentfs-releases/releases)，无需 GitHub 登录、Rust 或单独安装 jj。下面的命令可直接安装最新稳定版；重复执行即升级。

让 AI 配置 MCP、创建 workspace 并开始任务，请提供[公开 AI 集成指南](https://github.com/meglinge/agentfs-releases/blob/main/AI_INTEGRATION.md)。

默认入口为 `agentfsd stdio`；配置使用 `args = ["stdio"]`，工具用 repo/workspace_id 选择目标。v0.3.3 沿用 v0.3.1 的 IPC protocol 4 和全局配置，不连接旧 protocol 3 daemon。升级时仍须先停止旧 daemon，换新程序后重新连接 MCP，才能加载本地 Git 自动同步实现。

## Windows x64：PowerShell

在需要运行 AgentFS 的当前 PowerShell 中执行：

```powershell
irm https://github.com/meglinge/agentfs-releases/releases/latest/download/install.ps1 | iex
```

默认安装到 `%LOCALAPPDATA%\AgentFS\bin`，写入当前用户的持久 PATH，并更新当前 PowerShell 的 `$env:Path`；无需管理员权限，随后直接运行 `agentfsd --version`。支持 Windows PowerShell 5.1 和 PowerShell 7。

指定版本或安装目录：

```powershell
$env:AGENTFS_VERSION = 'v0.3.3'
$env:AGENTFS_INSTALL_DIR = 'D:\Tools\AgentFS\bin'
irm https://github.com/meglinge/agentfs-releases/releases/latest/download/install.ps1 | iex
```

恢复默认升级行为时删除这两个环境变量。已有 daemon 占用程序时，先用 `agentfsd stop --repo <仓库目录>` 停止相应进程再升级。

## Linux / macOS：Bash 或 Zsh

在当前 Bash/Zsh 中执行（需要 `curl`、`tar`、`sha256sum` 或 `shasum`）：

```bash
source <(curl -fsSL https://github.com/meglinge/agentfs-releases/releases/latest/download/install.sh)
```

默认安装到 `~/.local/bin`，更新当前 shell 的 PATH，并幂等写入 Bash 的 `.bashrc` 和有效的登录配置文件，或 Zsh 的 `.zshrc` / `.zprofile`（尊重 `ZDOTDIR`）。必须使用 `source`；通过 `curl | sh`、子进程 `bash` 或另开 PowerShell 执行，无法更新父终端的环境变量。

指定版本或目录：

```bash
export AGENTFS_VERSION=v0.3.3
export AGENTFS_INSTALL_DIR="$HOME/tools/agentfs/bin"
source <(curl -fsSL https://github.com/meglinge/agentfs-releases/releases/latest/download/install.sh)
```

只更新当前终端、不修改持久 PATH 时，设置 `AGENTFS_NO_MODIFY_PATH=1`。这些变量也支持 PowerShell。`AGENTFS_DOWNLOAD_ROOT` 可指定可信 HTTPS 镜像，镜像应提供相同的 `latest/download` 与 `download/<tag>` 路径结构。

## 平台与运行要求

| 平台 | 发布目标 | 运行要求 |
| --- | --- | --- |
| Windows x64 | `x86_64-pc-windows-msvc` | Windows 10 1809+ / Windows 11；静态链接 MSVC CRT |
| Linux x64 | `x86_64-unknown-linux-gnu` | glibc 2.35+（Ubuntu 22.04+）；不支持直接运行于 Alpine/musl |
| Linux ARM64 | `aarch64-unknown-linux-gnu` | glibc 2.35+ |
| macOS Intel | `x86_64-apple-darwin` | macOS 11+ |
| macOS Apple Silicon | `aarch64-apple-darwin` | macOS 11+ |

- Git fetch/push 需要 PATH 中有 `git`，并已设置可非交互使用的 Git 凭据。
- Windows 的虚拟 workspace 需要启用系统可选功能 Client-ProjFS，目录位于本地 NTFS。
- Linux 的虚拟 workspace 需要可挂载的 `/dev/fuse` 与 `fusermount3`。
- macOS 使用 local 工作副本；Windows/Linux 的虚拟后端不可用时也可使用 local。
- 安装器不修改系统可选功能。PATH 更新对当前执行安装命令的终端立即生效；已运行的编辑器、MCP 客户端和其他终端需重启，或配置可执行文件的完整路径。

## 安装验收与更新

安装命令结束后，在同一个终端运行：

```text
agentfsd --version
```

指定安装 `v0.3.3` 时应输出 `agentfsd 0.3.3`；未指定版本时，以最新稳定 Release 的 `VERSION` 为准。无需关闭当前终端，也无需额外执行 PATH 设置命令。升级前先停止使用旧程序的 daemon，升级后重新连接 MCP 客户端。

从 v0.3.1 升级后，调用 `workspace_open` 打开原目录的 `name: "default"`，或对已有 handle 调用 `workspace_status`，即可把新出现的本地 Git HEAD 同步为 parent；磁盘修改与暂存区保留。独立任务工作区不会自动移动基线，无需重新初始化 `.jj`。

维护者推送匹配 crate 版本的新 tag 后，流程执行 Windows/Linux 完整 CI 和五个平台的原生构建；全部成功并验证附件齐全后自动公开发布。

## 从 v0.2.x 或 v0.1.x 升级

1. 断开 MCP 客户端，使用仍然安装着的旧程序对各仓库执行 `agentfsd stop --repo <仓库目录>`；`--in-process` 会话通过退出客户端关闭。
2. v0.1.x 用户删除配置中的 HTTP 参数：`port`、`no_auth`、`allow_origin`、`session_idle_timeout`，以及客户端的 URL、bearer token 和 HTTP transport 配置。新版只接受 stdio；旧 daemon 的协议不能与新版混用。
3. 执行上面的一条安装命令，然后运行 `agentfsd client-config --client codex`，把生成的 stdio 配置合并到客户端；其他客户端替换 `--client`。默认参数只保留 `stdio`，移除固定的 `--repo`、`--workspace` 和 `--task-session`。保留其他 MCP 服务和必要的权限设置。
4. 重新连接，调用 `workspace_open({"repo":"实际仓库绝对路径","name":"原工作区名称"})`，核对返回的仓库、路径和基线。后续调用传入返回的 `workspace_id`。沿用原有 `.jj` 和工作区，无需重新初始化或删除项目数据。

本次移除 `read_file/list_dir/search/write_file/edit_file/move_path/delete_path` 与 `files` 工具组，无旧名称别名。文件编辑使用客户端原生工具；查询改用 `read_revision_file/list_revision_tree/search_revision_tree` 并显式指定 `rev`。原生修改后先调用 `workspace_status`，再搜索或比较 `@`。新的 `tree_move/tree_remove` 用于投影目录和异常路径的版本树修复。完整调用方式见[AI 集成指南](https://github.com/meglinge/agentfs-releases/blob/main/AI_INTEGRATION.md)。

## 校验、预发布与卸载

安装器先读取稳定版本号，再按固定 tag 下载对应压缩包与 `SHA256SUMS`，校验 SHA-256、归档内容和二进制版本后替换程序。下载或校验失败会保留已有程序。SHA-256 检测下载完整性，不代替独立的代码签名。

预发布版本不会替换最新稳定版。安装预发布版本时，设置 `AGENTFS_VERSION` 为对应 tag，再执行上述安装命令。

已发布版本的附件保留发布时快照；最新安装说明见[公开仓库首页](https://github.com/meglinge/agentfs-releases#readme)。

卸载前停止使用该程序的 daemon，删除安装目录中的 `agentfsd.exe` / `agentfsd`，再从用户 PATH 或 shell 配置中删除带 `# AgentFS` 的行。项目目录中的 `.jj` 和 workspace 保留。
