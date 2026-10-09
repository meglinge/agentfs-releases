# 安装 AgentFS

二进制和安装脚本公开发布在 [meglinge/agentfs-releases](https://github.com/meglinge/agentfs-releases/releases)，无需 GitHub 登录、Rust 或 jj。下面的命令可直接使用；重复执行即升级到最新稳定版。

## Windows x64：PowerShell

在需要运行 AgentFS 的当前 PowerShell 中执行：

```powershell
irm https://github.com/meglinge/agentfs-releases/releases/latest/download/install.ps1 | iex
```

默认安装到 `%LOCALAPPDATA%\AgentFS\bin`，写入当前用户的持久 PATH，并更新当前 PowerShell 的 `$env:Path`；无需管理员权限，随后直接运行 `agentfsd --version`。支持 Windows PowerShell 5.1 和 PowerShell 7。

指定版本或安装目录：

```powershell
$env:AGENTFS_VERSION = 'v0.1.0'
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
export AGENTFS_VERSION=v0.1.0
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

## 校验、预发布与卸载

安装器先读取稳定版本号，再按固定 tag 下载对应压缩包与 `SHA256SUMS`，校验 SHA-256、归档内容和二进制版本后替换程序。下载或校验失败会保留已有程序。SHA-256 检测下载完整性，不代替独立的代码签名。

预发布版本不会替换最新稳定版。首次只有预发布版本、尚无稳定版时，可从指定 Release 下载 `install.ps1` / `install.sh`，设置 `AGENTFS_VERSION` 后在当前 shell 中执行。

卸载前停止使用该程序的 daemon，删除安装目录中的 `agentfsd.exe` / `agentfsd`，再从用户 PATH 或 shell 配置中删除带 `# AgentFS` 的行。项目目录中的 `.jj` 和 workspace 保留。
