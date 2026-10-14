# 仓库、工作区和 MCP 路由

本文对应 v0.3.1。默认 `agentfsd stdio` 不绑定仓库，AI 在工具调用时选择项目。旧版本用户先按[安装指南](INSTALL.md)升级并重新生成客户端配置。

## 路由模型

```text
一个 MCP 连接（agentfsd stdio）
  ├─ repo A → daemon A → workspace_id A1 / A2
  └─ repo B → daemon B → workspace_id B1
```

启动全局 MCP 不创建工作区、不推断当前目录。首次选择 repo 时才发现或启动它的 daemon。每个 workspace_id 固定指向一个仓库中的工作区；没有可被其他调用切换的全局“当前仓库”。

| 对象 | 如何指定或核对 |
| --- | --- |
| 仓库 | open/create/list 的 repo 参数，必须是绝对 Git/jj 工作目录 |
| 工作区 | open 的 name；省略时同一连接内每个仓库使用一个稳定自动名称 |
| 后续调用目标 | 返回的 workspace_id，其他工作区工具必须明确提供 |
| 原生目录 | 返回的 path，客户端编辑器/终端必须使用这个路径 |
| 基线 | repository.git_head/git_branch、parents、working_copy 和 hint |

## 推荐调用顺序

1. 客户端只配置 `agentfsd stdio`，完成 MCP 初始化并读取 tools/list。
2. 调用 `workspace_open({"repo":"D:/Project3D","name":"task-a"})`。
3. 核对返回的仓库、路径、父提交、修改和提示，保存 repo/name/workspace_id 与任务起始 commit。
4. 用原生工具在 path 工作；后续调用如 `workspace_status({"workspace_id":"…"})`。
5. 可以继续打开其他 repo 或其他名称；返回的标识分别使用，互不改变。

已有工作区原生修改会被快照并保留。提供 base 时，新建使用它；复用必须与已有父提交集合匹配，否则拒绝。省略 base 沿用 workspace_create 的默认基线规则，不自动获取远端最新版本。不同于 Git HEAD 时返回提示，不会自动 rebase。

## 重试、并发与只读

同一 repo/name 的并发打开共用一条路由，不重复创建。省略 name 也有稳定候选名称；若要同仓库并行任务，应传不同 name。不同仓库可以使用相同工作区名称，返回不同 workspace_id。

daemon 重启后路由恢复到同一仓库和工作区。完整客户端重启后旧标识失效，显式提供保存的 repo/name 重新打开；标识不是长期存储的仓库编号。

workspace_open 结果未知或取消后重试相同 repo/name。已发送的普通修改不会自动重放，应检查 status、diff 和 op_log。工作区创建成功而后续操作失败时，保留已有数据供重试。

只读连接用 `workspace_list({"repo":"D:/Project3D"})` 获取已有工作区及标识。该调用不创建或快照工作区；打开尚未使用过的 Git 仓库仍可能按现有机制初始化 .jj。只读是 MCP 操作限制，不是文件系统沙箱。使用工具组时保留 workspace，例如 `--tools workspace,tree,history --read-only`。

多个客户端显式恢复同一个 repo/name 会共享该工作区，并非独占租约；并行编辑应使用不同名称。

## 原位布局与子模块

```text
D:/Project3D/
├── .git/                     # 也可为 Git 指针文件
├── .jj/agentfs/workspaces/
│   ├── task-a/
│   └── task-b/
└── 原有项目文件               # default 工作区
```

Git 原目录成为 default，.jj 与 .git 同级，无需另建管理目录。子模块和 Git linked worktree 的 .git 指针文件按 Git 规则解析。

修改子模块 D:/Project3D/engine 时直接调用：

```json
{"repo":"D:/Project3D/engine","name":"engine-task"}
```

不修改 MCP 配置，不移动或删除子模块。外层 gitlink 可能落后于子模块当前 HEAD，不能直接作为“当前实现”基线。

## 错误与专用连接

repo 缺失、相对路径、普通非仓库目录、未知 workspace_id 或旧作用域参数会明确拒绝。参数错误不会自动改连另一个仓库。仓库自身的配置、锁和权限分别生效；某个仓库失败不会改变其他标识。

全局工具使用 repo / workspace_id；不使用 expected_repo，也不靠省略作用域切换目标。专门限制到单仓库时，CLI 仍支持 --repo 加 --workspace、--task-session 或 --integrator；这些专用连接有不同的作用域 Schema，以其 tools/list 为准。普通多项目使用默认全局配置。

stdio stdout 只承载协议，stderr 承载日志。程序化集成使用正式 SDK，保持连接，处理取消、审批、超时和业务错误；日常 AI 调用复用已经配置的 MCP。
