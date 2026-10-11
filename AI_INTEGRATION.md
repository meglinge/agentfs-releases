# AgentFS：AI 快速集成指南

本文对应 v0.2.0，共 39 个 MCP 工具。安装命令与旧版升级步骤见 [安装说明](INSTALL.md)，布局与基线核对见 [工作区说明](WORKSPACE_SETUP.md)。

## 接入原则

AgentFS 管理独立工作区与版本；使用客户端原生文件工具编辑、移动、删除，并用原生终端运行普通检查。仓库范围搜索及版本查询优先使用快速 MCP 接口；磁盘上尚未快照的内容用原生工具读取、搜索。

对外仅 MCP stdio。多个客户端共享同一个仓库 daemon，内部使用本机 IPC。不要自行组装 HTTP 请求或实现另一套 JSON-RPC 生命周期；程序化接入使用正式 MCP SDK，保持会话，设置超时并消费 stderr。

## 配置

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

也可以生成配置：

```powershell
agentfsd client-config --client codex --repo D:/Project3D --workspace agent-a
```

同一个固定连接不会随聊天自动改变 workspace。并行任务分别使用 `agent-a`、`agent-b` 等名称和独立会话。已存在的正确工作区应复用，保留未提交修改。

编排者用 `--integrator` 替代 `--workspace`。它可显式指定工作区；普通工作会话绑定单个工作区，不能修改其他工作区。配置文件的 `[stdio]` 也可设置 `integrator = true`，此时不同时设置 workspace。

## 第一次连接

1. 完成 MCP initialize 并读取 instructions、tools/list。不要把内部 daemon protocol 3 当作 MCP 协议版本。
2. 调用 `workspace_status({})`。只在 `workspace_not_found` 时准备创建；认证、路径或绑定错误应先修正。
3. 调用 `workspace_list({})` 核对 `repository.path`、`git_head`、`git_branch`。外层容器仓库与子模块实现仓库是两个仓库，需要不同连接。
4. 如需创建，选择核对过的 base 调用 `workspace_create`；返回后再次核对 `path`、`parents`、`hint`。
5. 记录开始时的 commit，设置原生工具的实际工作目录为返回的 path。MCP 配置本身不会改变当前目录。

不要把子模块 gitlink 当成普通目录递归读取，也不要移动或删除子模块来修复读取错误。不要未经核对就用外层提交里较旧的子模块指针作为基线。

## 日常工作

- 通过客户端原生 edit/write 修改实际工作区文件。仓库范围查询用 `search_revision_tree`；需要历史内容用 `read_revision_file/list_revision_tree`，磁盘实时内容仍可用原生工具。
- 自动快照默认在文件停止变化约两秒后记录修改。status、版本修改操作与受控 run 也会记录相关修改；审阅前调用 status 获取当前事实。
- `workspace_status.changes` 相对当前 change 的父版本，最多列出 200 项。注意 `truncated` 与总数。
- 用 `diff` 检查工作区；跨多个 change 的任务必须比较任务开始的固定 commit 和结束版本。不要把当前 `@` 的 diff 当成整项任务修改列表。
- 原生工具记录编辑操作；客户端界面能否把其他进程写入、jj 版本和独立工作区全部显示出来取决于客户端。用版本差异补充核对。

## 版本树操作

| 工具 | 适用场景 |
| --- | --- |
| `read_revision_file` | 不切换工作区，读取指定 revision 的文件 |
| `list_revision_tree` | 不切换工作区，查询指定 revision 的目录树 |
| `search_revision_tree` | 正则或字面量检索指定版本，直接读取版本存储，避免逐个读取投影文件 |
| `annotate` | 查看每行来源与对应 change |
| `tree_move` | 移动 jj tree 中的跟踪路径，如原生工具无法移动的 ProjFS 投影目录 |
| `tree_remove` | 从 jj tree 移除异常/不可表示路径；保留未跟踪文件 |

读取示例：

```json
{"path":"src/lib.rs","rev":"@-"}
```

目录查询示例：

```json
{"rev":"main","path":"src","recursive":true,"limit":500}
```

`rev` 必填且不能为空；明确的 `@` 表示已快照的工作区版本。读取磁盘上的当前内容应使用原生工具。分页时沿用返回的 `next_offset` 与 `next_column`。

快速搜索示例（原生编辑后先调用 `workspace_status`）：

```json
{"pattern":"TODO|FIXME","rev":"@","paths":["src","!src/generated"],"context":2,"limit":100}
```

使用 `literal: true` 搜索原样文本，`ignore_case: true` 忽略大小写，`files_only: true` 只返回匹配文件和计数。路径支持目录、glob 和 `!` 排除规则。结果上限为 1,000 项，`truncated` 表示未全部返回；完整匹配计数仍保留，可缩小路径或条件继续查询。二进制、超过 32 MiB 的文件和忽略文件会报告跳过计数。

搜索只覆盖所选 revision 的文件。`include_ignored: true` 只包含版本树中被其 .gitignore 规则忽略的文件，不会读取尚未记录的磁盘文件。搜索不会代替快照，也不会修改文件或版本历史。可用 `--tools tree --read-only` 暴露只读查询。

ProjFS 投影目录原生重命名可能返回 `ERROR_NOT_SUPPORTED`，此时使用 `tree_move`。普通文件编辑、删除和可用的重命名操作不需要 MCP 替代品。

旧 `read_file/list_dir/search/write_file/edit_file/move_path/delete_path` 和工具组 `files` 已删除，无兼容别名。当前工具组为 `workspace,tree,history,changes,collab,exec,publish`。

## 多工作区与版本交付

每个工作区有自己的 `@`。读取其他工作区的版本可显式用 `other@`；修改其他工作区需要其绑定会话或集成者会话。准备集成时先核对各工作区 status 和 parents，再用 `integrate` 等工具组合，处理 conflicts 并运行检查。

整理完成的 change 后可 `new_change` 继续下一项工作。发布完成的 change 时应确认目标：新建了空 `@` 后通常发布 `@-`。fetch、rebase、bookmark 与 push 的参数以 tools/list 为准。清理工作区之前检查是否有待保留的工作和未跟踪文件。

## 错误与断线

同时检查 MCP `isError`、错误元数据和工具业务结果；例如外层执行成功并不意味着 `run.exit_code == 0`。

stdio 的 stdout 只承载协议，stderr 承载日志。daemon 断线后代理会重建连接和初始化。已知只读请求可以重试；已发送但响应丢失的写请求不会自动重放，会返回结果未知。先核对 workspace_status、diff 与 op_log，再决定后续动作。

工具组和 read-only 限制通过 MCP 元数据传递，initialize 必须确认。会话只能缩小 daemon 的权限。破坏性操作的确认、elicitation 与超时保留；这些不替代客户端对原生文件/终端工具的审批规则。

## 可复制的任务指令

> 使用绑定到目标仓库的 AgentFS MCP 管理工作区和版本。先调用 workspace_status；仅在 workspace_not_found 时创建，并用 workspace_list 核对仓库路径、Git HEAD 和分支，创建后检查 parents。以返回的 path 为原生工具工作目录，保存任务开始时的固定 commit。编辑、移动、删除和检查使用原生工具；仓库范围搜索使用 search_revision_tree 并显式给 rev，原生修改后先用 workspace_status 更新快照。磁盘实时内容用原生工具查询；历史内容和 ProjFS/异常路径修复使用版本树工具。结束前记录磁盘修改，用 status 和固定基线 diff 审阅，检查截断、MCP 错误及命令退出码。不要操作其他会话的工作区，不要重复执行结果未知的写调用。
