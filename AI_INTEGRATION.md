# AgentFS：AI 快速集成指南

本文对应 **v0.4.0**，共 40 个 MCP 工具。默认入口支持同一连接访问多个本地仓库；使用[安装指南](INSTALL.md)中的一条命令安装或升级。

## 一次配置，调用时选仓库

```json
{"mcpServers":{"agentfs":{"command":"agentfsd","args":["stdio"]}}}
```

也可运行 `agentfsd client-config --client codex`；支持 claude-code、cursor、vscode 和 generic。配置不填写 repo、workspace 或 task-session，启动时不会因当前目录而初始化项目。

先调用 `workspace_open`：

```json
{"repo":"D:/Project3D","name":"render-task"}
```

返回 `workspace_id`、`name`、`path`、`repository`、`parents`、`working_copy`、`created`、`bound`、当前修改和 `timings_ms`。保存 repo、name、workspace_id 和开始时固定 commit，核对目标目录及版本基线。bound 表示这条工作区路由已绑定，不表示整个 MCP 连接只能访问它。

后续所有工作区操作明确给 `workspace_id`：

```json
{"workspace_id":"返回的标识"}
```

以上是 `workspace_status` 参数。不存在隐式“当前仓库”；打开其他仓库不会改变已有标识。可继续 `workspace_open({"repo":"D:/OtherProject","name":"render-task"})`，两个同名工作区具有不同标识。

## 参数与恢复

- `workspace_open/create/list` 使用绝对路径 `repo`；其他工具使用 `workspace_id`。
- repo 必须是现有 Git/jj 工作目录。子模块用自己的目录；不用外层容器的 gitlink 代替实现仓库。
- open 的 name 可省略：同一连接内，每个仓库使用一个稳定自动名称，便于重试；需要更多工作区时明确传不同名称。
- open 复用已有名称并保留修改。可选 base 指定新建基线；复用时必须与现有父提交集合一致。不自动 fetch；独立任务工作区保留既有基线。
- 从 v0.3.3 开始，本地 Git 引用自动导入；原目录的 default 工作区在本地 HEAD 变化后自动同步 parent，并快照保留磁盘修改和 Git 暂存区。新建默认基线也会先刷新原目录。无需调用 git_fetch、寻找 jj 可执行文件或创建远端来导入本地提交。已有独立工作区 parent 与 HEAD 不同可以是正常隔离，不能自动 rebase；详见 [本地 Git 同步](WORKSPACE_SETUP.md#本地-git-同步)。
- workspace_id 属于当前 MCP 连接。daemon 重启后继续有效；整个客户端重启后，使用保存的 repo/name 重新打开获取标识。
- open 响应丢失或取消后重试相同 repo/name；其他写操作先用 status、diff、op_log 确认实际结果。
- `workspace_list({"repo":"D:/Project3D"})` 为每个已有工作区返回可用标识，不创建或快照工作区。只读连接通过它开始工作；自定义工具组需包含 workspace。
- 通用入口不接受旧的 workspace/name 作用域参数或 expected_repo。name 仍可用于工具自身的业务参数，例如 bookmark_set.name。

## 原生工具与版本查询

AgentFS 管理工作区、版本和快速查询。文件编辑、移动、删除以及普通检查使用客户端原生工具；在返回的 path 中工作，MCP 不会自动改变原生工具的当前目录。

原生编辑后先 `workspace_status({"workspace_id":"…"})` 更新快照，再查询 rev @。查询历史可使用固定 commit，磁盘上尚未快照的内容用原生读取和搜索。

`search_revision_tree` 示例：

```json
{"workspace_id":"返回的标识","rev":"@","pattern":"TODO|FIXME","paths":["src"],"context":2,"limit":100}
```

`read_revision_file` 示例：

```json
{"workspace_id":"返回的标识","path":"src/lib.rs","rev":"@-"}
```

`list_revision_tree` 同样要求显式 rev。读取分页沿用 next_offset/next_column；搜索支持 literal、ignore_case、files_only、路径/glob 和忽略规则。搜索不会读取未进入版本树的磁盘文件。

Windows ProjFS 投影目录无法原生重命名时使用 tree_move；异常或不可表示的跟踪路径可用 tree_remove。两者也必须传 workspace_id。普通文件编辑不经 MCP，客户端可以记录原生编辑列表。

## 审阅、协作与边界

status 的 changes 相对当前 change 的父版本，最多 200 项；检查 truncated 和总数。整项任务应以开始时保存的固定 commit 与结束版本做 diff。跨多个 change 的任务不能只看当前 @。

### 子模块状态（v0.4.0）

从 v0.4.0 开始， `workspace_open/status` 增加 `submodules`：每项包含 `path`、完整 `recorded_commit`（当前快照 `@` 的 gitlink）、完整 `current_commit`（该工作区子模块自己的 HEAD）和 `state`。状态为 `unchanged`、`modified`、`uninitialized`、`unavailable` 或 `conflicted`；无法确定的提交字段省略，错误或冲突另有 `detail`。读取失败不能当成没有修改。

存在跟踪子模块时返回 `submodule_snapshot_supported: false` 和说明。`changes` 与 `diff` 仍只表示已快照内容；即使 `changes: []`，也必须检查 `submodules`。`unchanged` 只表示 HEAD 一致，不代表内部文件干净：检查不会扫描子模块文件，也不递归检查它的子模块。独立工作区只读取自己目录里的子模块，未初始化时不会借用原目录的 HEAD。

子模块列表最多 200 项，超过时返回 `submodules_total` 和 `truncated`；剩余磁盘状态用返回的工作区 path 下的原生 Git 检查，不能通过快照 diff 补齐。编辑子模块要独立打开其仓库；指针用原生 Git 在外层仓库记录并提交后，重新打开外层工作区自动导入。已经提交的 gitlink 支持历史 diff。见[详细说明](WORKSPACE_SETUP.md#子模块状态报告v040)。

同一仓库的工作区可通过 integrate/rebase/冲突工具协作。不同仓库有各自的 jj 图、锁和配置；不要把另一仓库的 change 或 workspace_id 当成本仓库 revision。

每个仓库按需启动或复用 daemon，只读和工具组选择继续生效；进度和取消沿路由转发。v0.4.0 已移除 AgentFS 自身的破坏性开关、confirm 参数和审批请求。推送、删除工作区、恢复操作及显式版本改写不需要额外确认参数；仓库/工作区边界、不可变历史、冲突和远端状态校验仍然执行。MCP 只用 stdio 和本机 IPC，不提供 HTTP。

同时检查 JSON-RPC error、MCP isError 和业务结果（如 run.exit_code）。全局 workspace_open 的 timings_ms.total 包含整个网关调用，connect 包含首次启动/连接目标 daemon，backend 是后端内总耗时；repository/open/snapshot 为后端阶段。不包括模型思考和启动 MCP 客户端的时间。

确需限定到单仓库时可显式使用 --repo 加 --workspace/--task-session/--integrator；这是专用连接，其参数以自身 tools/list 为准，不要与全局入口混用。

## 审批控制迁移（v0.4.0）

新接口不接受 confirm 参数；CLI/配置不再接受 allow_destructive、destructive_approval、approval_timeout 或对应命令行选项。删除这些参数后，用旧二进制停止已有 daemon，再替换程序并重新连接 MCP，以刷新工具 Schema。IPC 协议版本为 5，旧客户端或 daemon 不混用；版本有效性和 workspace_id 恢复规则不变。

普通调用示例：git_push({workspace_id, bookmark, remote})、workspace_remove({workspace_id})、op_restore({workspace_id, op})。读取 tools/list 的实际 Schema，不要向新版传旧版 confirm，升级后确认 stdio 和 daemon 都运行 v0.4.0。

## 可复制的任务指令

> 使用全局 AgentFS MCP，以 workspace_open 的 repo 参数选择实际项目；恢复任务时指定保存的 name。核对返回的 repository、path、parents 和 hint，保存 workspace_id、工作区名称和开始时固定 commit。后续工作区调用都传 workspace_id；并行任务使用不同名称，并保持标识与实际目录对应。原生编辑、移动、删除和检查在 path 中进行；快照后再查询 rev @。完成后用 status 和固定基线 diff 审阅，检查截断、工具错误及业务退出码。有子模块时检查 submodules 和 hint；旧版本没有该字段时用原生 Git 补查，不能把空 changes 当成磁盘无修改。不要修改 MCP 配置切换项目，也不要另建临时 SDK 绕过作用域。
