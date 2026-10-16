# 仓库、工作区和 MCP 路由

本文对应 v0.4.0。默认 `agentfsd stdio` 不绑定仓库，AI 在工具调用时选择项目。旧版本用户先按[安装指南](INSTALL.md)升级；全局 stdio 配置可沿用，但需删除旧审批选项，停止旧 daemon 并升级后重新连接 MCP。

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

已有工作区原生修改会被快照并保留。提供 base 时，新建使用它；复用必须与已有父提交集合匹配，否则拒绝。省略 base 沿用 workspace_create 的默认基线规则，不自动获取远端最新版本。独立工作区不同于 Git HEAD 时返回提示，保留原有基线。

## 本地 Git 同步

此功能从 v0.3.3 开始提供。

在原目录执行 `git commit`、`git switch` 或 `git reset` 后，下一次打开/访问原目录的 `default` 工作区会导入本地 HEAD，并把磁盘现有内容记录为新 HEAD 上的工作区修改。已经进入 Git 提交的改动不再重复显示。这个过程不写 Git 暂存区、不检出覆盖磁盘，也不需要远端、网络或 jj 命令行。

本地分支和 tag 在版本查询及工作区创建前自动导入；省略 base 创建任务时先刷新原目录的 HEAD，再选择基线。旧版 `.jj` 元数据首次使用时建立 HEAD 记录；已有 parent 正确时保留原工作提交。HEAD 不变时，主动选择的历史基线仍会保留；重复访问不新增同步操作。

其他独立工作区保持自己的 parent 和修改，不会因原目录有新提交而自动 rebase。`workspace_list` 可导入引用，但不快照或移动工作区；同步原目录请使用 `workspace_open` 的 `name: "default"`，或对其 handle 调用 `workspace_status`。仓库有外部命令持锁时，同步遵守原有等待/延后规则。

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

### 子模块状态报告（v0.4.0）

外层快照保存已有 gitlink，但不会自动记录子模块 HEAD 的移动，也不扫描子模块内部文件。从 v0.4.0 开始，在 `workspace_open` 和 `workspace_status` 增加独立报告。例如外层快照记录 A，而子模块已切换到 B：

```json
{
  "changes": [],
  "submodules": [{
    "path": "engine",
    "recorded_commit": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "current_commit": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "state": "modified"
  }],
  "submodule_snapshot_supported": false
}
```

`recorded_commit` 取自 `@`，不是 Git 暂存区；检查不会修改暂存区、子模块 HEAD 或文件。`state` 的含义：

| 状态 | 含义 |
| --- | --- |
| `unchanged` | 当前 HEAD 与快照指针一致；内部文件是否修改未检查 |
| `modified` | HEAD 与快照指针不同，尚未纳入外层快照或当前 diff |
| `uninitialized` | 子模块路径不存在，或没有自己的 `.git` |
| `unavailable` | 路径、Git 元数据或 HEAD 无法可靠读取；见 detail，不能视为干净 |
| `conflicted` | 快照中的 gitlink 有冲突；recorded_commit 字段省略 |

当前 HEAD 无法取得时省略 current_commit 字段。每次只检查当前工作区中已跟踪的 gitlink；独立工作区不会读取原目录的子模块，也不会向上发现父仓库冒充子模块。列表最多 200 项；截断时 submodules_total 给总数，原生 Git 可补查其余路径，快照 diff 无法补齐这些磁盘观察。

首次状态查询从版本树收集 gitlink；随后缓存按版本树差异更新，每次重新读取子模块 HEAD。不会递归遍历子模块文件、计算内部文件 diff、启动 Git 子进程或访问网络。

编辑子模块时打开子模块自身仓库。确认要更新外层指针后，在外层执行 `git add -- engine` 并提交选定修改，再重新打开外层工作区自动导入 Git 提交。历史版本中的 gitlink 差异一直可以由 AgentFS diff 展示；此次修复只补状态可见性，不改变快照和提交语义。

## 移除审批控制（v0.4.0）

AgentFS 不再提供 allow_destructive 总开关、destructive_approval 审批模式、approval_timeout 或工具 confirm 参数。显式调用直接执行；只读和工具组选择、仓库/工作区绑定、原目录保护、不可变历史与冲突校验、Git 远端状态检查仍然生效。MCP 的 destructiveHint 仍如实描述工具副作用，不作为 AgentFS 的审批门槛。

升级顺序：

1. 用旧程序停止各仓库 daemon。
2. 从 CLI 参数和配置中删除 --allow-destructive、--destructive-approval、--approval-timeout，以及对应的三个下划线配置键。
3. 更新二进制并重连 MCP，重新读取工具 Schema；调用时删除 confirm。
4. 使用保存的 repo/name 恢复 workspace_id。

旧审批选项会被明确拒绝，不能静默沿用。会话元数据仅保留 tools/readOnly；IPC 协议升级到 5，因此旧代理与新 daemon 不混用。v0.3.x 客户端和 daemon 必须一起升级，沿用已有 `.jj` 与工作区。

## 错误与专用连接

repo 缺失、相对路径、普通非仓库目录、未知 workspace_id 或旧作用域参数会明确拒绝。参数错误不会自动改连另一个仓库。仓库自身的配置、锁和权限分别生效；某个仓库失败不会改变其他标识。

全局工具使用 repo / workspace_id；不使用 expected_repo，也不靠省略作用域切换目标。专门限制到单仓库时，CLI 仍支持 --repo 加 --workspace、--task-session 或 --integrator；这些专用连接有不同的作用域 Schema，以其 tools/list 为准。普通多项目使用默认全局配置。

stdio stdout 只承载协议，stderr 承载日志。程序化集成使用正式 SDK，保持连接，处理取消、超时和业务错误；日常 AI 调用复用已经配置的 MCP。
