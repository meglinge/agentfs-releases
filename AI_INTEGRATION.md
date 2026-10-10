# AgentFS：AI 快速集成指南

适用：通过 MCP 使用 AgentFS 的 AI coding agent，以及为 agent 接入 MCP 的开发者。本文以 [v0.1.1](https://github.com/meglinge/agentfs-releases/releases/tag/v0.1.1) 为接入基线，包含原地初始化和一条命令安装。仓库示例为 `D:/Project3D`；工具名、参数和返回结构以实际安装版本的 `initialize` instructions 与 `tools/list` Schema 为准。

## 1. 先记住这几件事

子模块接入、仓库身份和基线诊断见[工作区接入检查](WORKSPACE_SETUP.md)，这些能力从 v0.1.1 开始提供。

- `--repo` 指向原来的 Git 工作目录。首次启动自动在 `.git` 旁创建 `.jj`，无需另建管理目录。
- 每个并行任务使用不同的 workspace 名称；多个客户端绑定同一个名称，就会修改同一份工作副本。
- agent 的实际工作目录取自 `workspace_create` 或 `workspace_status` 返回的 `path`，不要把原仓库根目录当作自己的 workspace。
- MCP 文件路径使用仓库内相对路径和 `/`，例如 `src/Scene.cpp`；绝对路径仅用于配置服务和设置外部编辑器的工作目录。
- `@` 表示当前 workspace 正在编辑的 change；文件修改自动记录，不需要 `git add` 或 `git commit`。
- 文件读取工具读的是已记录的版本树。外部编辑器刚改完文件时，先调用 `workspace_status` 再读取、比较或交付。
- 任务完成后报告 workspace、`path`、change id、测试结果；只有任务要求发布或清理时，才执行对应操作。

默认布局：

```text
D:/Project3D/
├─ .git/                         原 Git 仓库
├─ .jj/                          jj 元数据
│  └─ agentfs/
│     ├─ daemon/                 常驻进程状态
│     └─ workspaces/
│        ├─ agent-a/             任务 A
│        ├─ agent-b/             任务 B
│        └─ integration/         汇总与测试
└─ 原有项目文件                   default 工作副本
```

workspace 子目录在调用 `workspace_create` 后创建。初始化保留原文件、Git HEAD、分支和暂存区；磁盘上的未提交内容按忽略规则及新文件大小限制记录到 `default@`。原目录不参与后台自动 snapshot。

## 2. 接入 MCP 客户端

### 准备程序

先运行 `agentfsd --version` 检查是否已安装及实际版本；已有可用程序时直接配置 MCP。需要安装或升级时，从公开 Release 获取二进制，无需 GitHub 登录、源码仓库权限、Rust 或单独安装 jj。原地初始化支持普通 Git 工作目录，以及使用 `.git` 指针文件的子模块和 linked worktree，详见[工作区接入检查](WORKSPACE_SETUP.md)。

Windows x64，在当前 PowerShell 运行：

```powershell
irm https://github.com/meglinge/agentfs-releases/releases/latest/download/install.ps1 | iex
agentfsd --version
```

Linux x64/ARM64 或 macOS Intel/Apple Silicon，在当前 Bash/Zsh 运行：

```bash
source <(curl -fsSL https://github.com/meglinge/agentfs-releases/releases/latest/download/install.sh)
agentfsd --version
```

安装器校验文件和版本，持久保存用户 PATH，并让当前执行命令的终端立即生效。默认目录为 Windows 的 `%LOCALAPPDATA%\AgentFS\bin`、Linux/macOS 的 `~/.local/bin`。AI 工具新开的子进程无法更新其父进程或已经运行的 MCP 客户端的环境变量；此时用安装后的完整路径运行 `agentfsd client-config`，并采用生成配置中的可执行文件路径，或重启客户端。不要仅凭安装成功就假定当前 MCP 会话已经接入。

未设置 `AGENTFS_VERSION` 时安装最新稳定版；需要固定版本可将它设为 `v0.1.1`，并核对输出为 `agentfsd 0.1.1`。重复安装即升级，升级前先停止使用旧程序的 daemon，之后重新连接客户端。指定目录、运行要求和卸载见[安装说明](INSTALL.md)。仅在任务需要开发 AgentFS 本身时，才使用源码构建流程。

Windows 默认探测 ProjFS，Linux 默认探测 FUSE；不可用时，新 workspace 使用普通 `local` 工作副本。若必须使用虚拟工作副本，传 `--working-copy agentfs-v2`，探测失败会明确报错。平台准备见[安装说明的平台要求](INSTALL.md#平台与运行要求)。

### 首选 stdio

让 MCP 客户端启动以下程序；通常不用另外运行 `serve`：

```powershell
agentfsd stdio --repo D:/Project3D --workspace agent-a
```

第一个客户端会初始化 `.jj` 并启动后台 daemon；后续客户端共用它。后台 daemon 默认在无客户端 300 秒后退出。不要给并行客户端使用 `--in-process`，该模式独占仓库。

通用 `mcpServers` 配置示例，前提是客户端能从 `PATH` 找到 `agentfsd`：

```json
{
  "mcpServers": {
    "agentfs": {
      "command": "agentfsd",
      "args": ["stdio", "--repo", "D:/Project3D", "--workspace", "agent-a"]
    }
  }
}
```

第二个 agent 使用同样配置，只将 workspace 改成 `agent-b`。配置 MCP 不会自动切换客户端自身终端或编辑工具的工作目录。

也可以让程序生成对应客户端的配置，避免手写格式和可执行文件路径：

```powershell
agentfsd client-config --client generic --repo D:/Project3D --workspace agent-a
agentfsd client-config --client claude-code --repo D:/Project3D --workspace agent-a
agentfsd client-config --client codex --repo D:/Project3D --workspace agent-b
```

输出为待合入客户端现有配置的内容；保留其他配置项。`--client cursor`、`--client vscode` 也受支持。Codex 生成配置包含启动和调用超时；其他客户端也应让调用超时覆盖 `run.timeout_secs` 及返回结果所需时间。更多配置选项以 `agentfsd client-config --help` 为准。

### 需要 HTTP 时

先运行固定端口的服务：

```powershell
agentfsd serve --repo D:/Project3D --port 8765
agentfsd client-config --client generic --transport http --repo D:/Project3D --workspace agent-a
```

连接使用 Streamable HTTP，必须携带生成配置中的认证信息。含 token 的配置不要提交到版本库；可以使用生成器的 `--token-env` 选项引用环境变量。

| 接入方式 | 会话范围 |
|---|---|
| stdio `--workspace agent-a` | 绑定 `agent-a` |
| HTTP `/mcp/agent-a` | 绑定 `agent-a` |
| HTTP `/mcp` | 集成者会话，可显式选择操作的 workspace |

绑定会话只能在自己的 workspace 中操作，但可用 `rev: "agent-b@"` 读取其他 change。集成者会话对一般工具传 `workspace`；`workspace_status`、`workspace_remove` 使用 `name`。

## 3. 连接后的协议约定

使用 MCP SDK 完成 `initialize`、`notifications/initialized`，再获取 `tools/list`。读取服务返回的 instructions，并按会话实际暴露的工具工作；`--tools` 和 `--read-only` 可能缩小工具集合。

下文的 JSON 块是 **`tools/call` 的 params**，不是 shell 命令。若客户端直接提供工具调用接口，将 `name` 和 `arguments` 分别交给该接口即可。客户端显示的工具名可能带服务器前缀，以发现结果为准。

原始 JSON-RPC 调用的完整形式如下，`id` 由客户端分配：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "workspace_status",
    "arguments": {}
  }
}
```

stdio 的 stdout 是 MCP 协议流，日志在 stderr；不要向协议流混入日志。HTTP 会话和重连由 MCP SDK 管理。AgentFS daemon protocol 2 是内部进程发现协议，不能拿它替代 MCP 协商的协议版本。

## 4. AI 的最短工作流程

下面假设会话绑定 `agent-a`。示例中的文件内容仅用于展示参数，实际修改前先读取目标文件。

### 第一步：取得工作目录

```json
{"name":"workspace_status","arguments":{}}
```

仅当错误码为 `workspace_not_found` 时才需要创建。创建前先检查实际连接的仓库：

```json
{"name":"workspace_list","arguments":{}}
```

核对 `repository.path`、`git_dir`、`git_head` 和 `git_branch`；容器与子模块是不同仓库，不能未经核对把容器记录的旧 gitlink 当作基线。确认后创建本会话对应的 workspace：

```json
{"name":"workspace_create","arguments":{"name":"agent-a"}}
```

未指定 `base` 时，优先从 `default` 的父提交开始；原地初始化时这通常是初始化时的 Git HEAD。需要固定基准时，传入已经确认的 bookmark、change id 或 commit id，不要猜测仓库一定有 `main`。

保存结果中的 `path` 和 `working_copy`，核对 `parents` 与 `hint`。创建基线不同于 Git HEAD 时会提示，但不会自动切换历史基线。复用已存在的 workspace 时先检查其改动，不要重新创建、清空或撤销已有工作。

### 第二步：读取和定位

```json
{"name":"list_dir","arguments":{"limit":100}}
```

```json
{"name":"read_file","arguments":{"path":"README.md","limit":200}}
```

```json
{"name":"search","arguments":{"pattern":"TODO|FIXME","paths":["src"],"limit":50}}
```

文件不存在时，依据目录列表调整路径。优先读取项目自己的开发说明和任务涉及的文件。

### 第三步：修改

整份写入：

```json
{"name":"write_file","arguments":{"path":"docs/task-notes.md","content":"Task notes\n"}}
```

精确替换：

```json
{"name":"edit_file","arguments":{"path":"docs/task-notes.md","old":"Task notes\n","new":"Verified task notes\n"}}
```

`old` 必须精确匹配，默认只能出现一次；不要为了消除歧义直接打开 `replace_all`。也可以用外部编辑器修改返回的 `path` 内的文件。完成外部编辑后调用 `workspace_status`，让后续 MCP 读取看到最新内容。

Windows ProjFS workspace 的目录移动使用 `move_path`。版本管理通过 MCP 工具完成；这些目录不是独立的 Git worktree，在其中调用普通 Git 可能向上发现原仓库，不能用它代替本 workspace 的版本操作。不要手动编辑 `.jj` 元数据。

### 第四步：运行检查

先确认 shell 的当前目录，例如 Windows：

```json
{"name":"run","arguments":{"command":"Get-Location","timeout_secs":30}}
```

Linux 对应命令为 `pwd`。随后把 `command` 替换为从项目说明中确认的构建、测试或 lint 命令；`cwd` 可指定 workspace 内的相对子目录，省略时就是根目录。

`run` 默认超时 120 秒，允许 1–3600 秒。检查返回的 `exit_code`、`timed_out`、`cancelled`、`stderr`、`snapshot_error` 和 `untracked`。收到 MCP 成功响应不代表测试退出码为零，也不代表所有新文件都已进入版本历史。

### 第五步：审阅和交付

```json
{"name":"diff","arguments":{"format":"summary"}}
```

```json
{"name":"diff","arguments":{"format":"unified","paths":["docs/task-notes.md"]}}
```

确认任务完成后描述当前 change：

```json
{"name":"describe","arguments":{"message":"Document verified task notes"}}
```

如需保留已完成 change，并打开下一项工作的空 change：

```json
{"name":"new_change","arguments":{}}
```

调用前完成的成果在 `@`；调用后它成为 `@-`。交付时优先报告已完成成果的 change id，避免后续 `@` 移动后引用错位置。不要把 jj operation id 当成 change id。

## 5. 从原目录的未提交修改开始

默认创建 workspace 不包含 `default@` 自身的未提交修改。需要包含时：

1. 由绑定 `default` 的准备会话，或 HTTP `/mcp` 集成者会话，先记录原目录当前状态。
2. 再让各任务会话从 `default@` 创建自己的 workspace。

准备会话调用：

```json
{"name":"workspace_status","arguments":{"name":"default"}}
```

绑定 `agent-a` 的会话调用：

```json
{"name":"workspace_create","arguments":{"name":"agent-a","base":"default@"}}
```

`agent-a` 的绑定会话不能自行调用 `workspace_status(name="default")`。需要多任务完全一致的初始内容时，协调者把准备阶段返回的 `working_copy.commit_id` 作为固定 `base` 分发，避免创建期间原目录又被修改。

## 6. 多 workspace 汇总

每个 agent 连接相同 `--repo`，使用不同 `--workspace`。任务结束后先停止继续编辑已交付的 change，再由绑定 `integration` 的第三个会话汇总。

第三个会话先创建自己的 workspace：

```json
{"name":"workspace_create","arguments":{"name":"integration"}}
```

下面假设基础 bookmark `main` 已确认存在，并且两个任务都在完成后调用了 `new_change`：

```json
{
  "name":"integrate",
  "arguments":{
    "changes":["agent-a@-","agent-b@-"],
    "onto":"main",
    "message":"Integrate agent A and agent B"
  }
}
```

实际集成优先使用各 agent 报告的已完成 change id。若 agent 没有调用 `new_change`，它的成果还在 `agent-a@`，不能直接套用 `@-`。`onto` 换成共同约定的基准。

`integrate` 创建汇总 change，不直接改写各 agent 的源 change。检查返回的冲突，再运行项目测试。冲突处理先读取详情：

```json
{"name":"conflicts","arguments":{}}
```

```json
{"name":"conflict_details","arguments":{"path":"src/Scene.cpp","full":true}}
```

自行合并后，按返回的 `token`、`hint` 和实际内容调用 `resolve_conflict`；片段 token 只能配对应片段的合并结果。仅在已经读齐整份文件并确认结果时，使用 `write_file` 写完整合并内容。重新检查冲突与测试结果。

绑定 `integration` 的会话无需传 `workspace`；若通过 HTTP `/mcp` 操作，上述 `integrate`、`conflicts`、文件工具及 `run` 调用需补 `workspace: "integration"`。

## 7. 发布与清理是可选步骤

普通接入默认不允许破坏性工具。任务明确需要发布或删除 workspace 时，服务端和对应会话都必须允许破坏性调用（`--allow-destructive`），调用还必须带 `confirm: true`。已有 daemon 的策略不匹配时按错误提示协调配置，不能靠重试扩大权限。

例：在集成会话中发布当前已验证的成果；以下只示范参数，不属于每次任务都应执行的步骤：

```json
{"name":"bookmark_set","arguments":{"name":"feature/combined-work","rev":"@"}}
```

```json
{"name":"git_push","arguments":{"bookmark":"feature/combined-work","remote":"origin","confirm":true}}
```

如果已经调用 `new_change`，bookmark 应指向已完成的 change id 或 `@-`。`origin` 也必须是仓库实际存在且已配置认证的 remote。

经确认不再需要 workspace 后，在对应绑定会话或集成者会话调用：

```json
{"name":"workspace_remove","arguments":{"name":"agent-a","confirm":true}}
```

此调用会忘记该 workspace 并删除其目录。原目录的 `default` 是仓库锚点，不能这样删除。不要用递归文件删除代替 workspace 管理工具。

`--destructive-approval user` 在客户端支持 MCP elicitation 时还会询问用户。不支持该能力的客户端依照 `confirm` 规则执行，服务 instructions 会说明；需要彻底禁止这些操作时不启用 `--allow-destructive`。无人值守会话使用用户批准模式可能等待至超时。

## 8. 程序化处理结果

- 检查 JSON-RPC 是否返回 `error`；若正常收到工具结果，再检查 `result.isError`。
- 业务数据优先读 `result.structuredContent`；`content` 提供文字摘要。错误码位于 `result._meta["agentfs/errorCode"]`。
- 未提供的可选参数通常应省略；不要拼入 Schema 没有声明的字段。Schema 是当前会话的参数依据。
- 返回结果可能省略 false、空列表或可选字段；按各工具的输出 Schema 和字段语义处理，不能把任何缺失字段都统一当作 false。
- `truncated: true` 表示结果不完整，继续读取或缩小范围。`read_file` 用 `next_offset` 续读，返回 `next_column` 时同时传 `column`。
- 文本 offset 是 1 起始的行号；二进制 offset 是 0 起始的字节数。二进制内容为 base64，不要直接拼入文本编辑工具。
- 保留并处理 `hint`、`notice`、冲突及未记录文件信息。文件变更后出现的路径更新提示不能忽略。
- 对已发送但响应丢失的修改调用，先检查状态、diff 或操作日志，再决定是否重试；重复 `new_change`、`edit_file` 或发布可能产生新的结果。

常见错误的下一步：

| 错误码 | 处理 |
|---|---|
| `workspace_not_found` | 核对 `workspace_list.repository` 和基线后创建绑定的 workspace |
| `workspace_exists` | 检查并复用已有 workspace，或为新任务换名 |
| `workspace_not_allowed` | 修正会话绑定或交由集成者操作 |
| `base_required` / `invalid_revision` | 查询实际历史，传入有效且明确的基准 |
| `tool_not_exposed` | 核对 `tools/list` 与会话配置 |
| `old_text_not_found` / `old_text_ambiguous` | 重新读取文件，修正精确替换范围 |
| `confirmation_required` / `destructive_disabled` | 核对任务授权与服务策略，不盲目追加确认或重试 |
| `user_declined` / `approval_failed` | 处理拒绝或审批失败，不自动重复触发 |
| `conflict_changed` | 重新读取冲突详情，再生成合并结果 |
| `push_rejected` | 根据提示 fetch、检查远端变化并协调合并 |

诊断服务：

```powershell
agentfsd status --repo D:/Project3D
```

日志位于 `.jj/agentfs/daemon/daemon.log`。一个仓库只运行一个 daemon；更改共同的仓库选项需要协调其他客户端。当前工作副本只接受完整 version 2 状态，不提供旧格式迁移；升级前结束旧服务。

## 9. 可直接交给 AI 的工作指令

将以下内容中的 workspace 名称和任务替换为实际值；前提是客户端已连接对应 MCP 服务。

```text
你通过 AgentFS MCP 完成任务。
仓库：D:/Project3D
分配的 workspace：agent-a
任务：<填写任务目标和验收条件>

先读取当前 MCP instructions 和工具 Schema，确认会话绑定。
调用 workspace_status；仅在 workspace_not_found 时需要创建 agent-a。
创建前调用 workspace_list，核对 repository 的路径、Git HEAD 和分支，再确定 base。
创建后核对 parents 和 hint，确认基线符合任务；不要盲用容器记录的旧子模块提交。
以返回的 path 作为实际工作目录，读取该 workspace 内的项目说明。
保留已有改动，所有本任务文件编辑与测试均在此 workspace 完成。
外部编辑完成后调用 workspace_status，再使用 diff 审阅。
根据项目说明选择测试命令，通过 run 执行并检查退出码与记录结果。
任务完成后用 describe 描述成果；若调用 new_change，记住成果已变成 @-。
交付 workspace 名称、path、成果 change id、修改摘要、测试结果和未解决事项。
仅在任务要求且有相应授权时进行发布、删除或改写他人的 change。
```

安装与升级见[安装说明](INSTALL.md)；接入选项可用 `agentfsd stdio --help`、`agentfsd serve --help` 和 `agentfsd client-config --help` 查询。完整工具列表、参数定义与返回结构由当前会话的 `tools/list` 提供；使用二进制接入无需访问私有源码。
