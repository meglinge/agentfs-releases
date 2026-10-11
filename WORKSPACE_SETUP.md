# 仓库、会话与工作区接入检查

本文对应 v0.2.0。Git 指针文件、仓库诊断和原位布局继续保留；文件修改使用客户端原生工具；搜索、读取和目录查询可走显式 revision 的快速 MCP 接口，MCP 接口以当前版本的 instructions 和 tools/list 为准。

## 确认仓库、会话和基线

| 对象 | 需要确认的内容 |
| --- | --- |
| 仓库 | `--repo` 是本次要修改的代码仓库根目录；容器及其子模块是不同仓库 |
| 会话 | `initialize` 的 instructions 绑定本任务的 workspace；并行任务使用独立名称 |
| 基线 | 工作区 `parents` 与任务要求的当前 Git HEAD、目标分支或历史版本对应 |

配置中固定的 `--workspace` 不会因为新开聊天而自动变化。绑定会话也不能通过工具的 `workspace` 参数切换到其他工作区。明确目标仓库和任务名称后，用 `agentfsd client-config` 生成连接配置，并由客户端加载。

## 推荐调用顺序

1. 完成 MCP `initialize` 和 `notifications/initialized`，读取 instructions 及 `tools/list`，确认绑定。
2. 调用 `workspace_status`。已有工作区先审阅路径、父提交和改动；仅 `workspace_not_found` 表示需要创建。
3. 创建前调用 `workspace_list`，核对顶层 `repository.path`、`repository.git_dir`、`repository.git_head` 和 `repository.git_branch`。仓库不匹配时先连接正确仓库。
4. 根据任务确定基线；省略 `base` 使用规定的默认值，固定基线时传已核对的完整 commit id。`git_head` 是读取时的 Git 状态，可能尚未导入 jj，不代表可以跳过 revision 查询或导入检查。
5. `workspace_create` 后核对 `parents`、`path` 与 `hint`。父提交与 Git HEAD 不同会返回提示，历史版本任务仍允许继续。相同也不意味着它是远端最新版本，目标分支及同步情况仍由任务决定。
6. 以返回的 `path` 为实际目录；设置原生工具的实际工作目录，读取 `AGENTS.md` 等说明，然后搜索、编辑和测试。

`repository` 同时提供结构化字段和文本内容，用于识别连接目标。该检查不会自动 fetch、切换分支、更新旧 workspace 或合并其他任务的改动。

## 子模块与 Git worktree

AgentFS 接受普通 Git 工作目录、子模块及 linked worktree 的 `.git` 指针文件；`--init-from-git` 的来源也可使用明确给出的 Git 元数据目录或 bare repository。相对指针按 Git 规则解析；linked worktree 保留自己的 HEAD。解析只检查传入的位置，不向上查找另一个仓库。

例如外层 `D:/Project3D` 中的 `engine/` 是子模块，修改 engine 时使用：

```powershell
agentfsd client-config --client generic --repo D:/Project3D/engine --workspace engine-task
```

首次启动后 `.jj` 位于子模块工作目录中，与 `.git` 指针文件同级；无需手工寻找 `.git/modules/...` 或另建管理目录。原有 Git HEAD、暂存区和文件内容保持，未提交内容按 jj 忽略规则记录在 `default` 中。

外层提交记录的 gitlink 可能落后于子模块当前检出的版本，不能直接当作“当前实现”的基线。更新子模块后，应提交外层的 gitlink；已经建立的旧工作区仍需单独核对。

从外层工作区读取或遍历子模块时，工具会提示连接子模块仓库。读取子模块无需删除或移动它。

## 错误处理与客户端实现

| 情况 | 下一步 |
| --- | --- |
| `workspace_not_found` | 确认仓库和基线后创建本会话绑定的工作区 |
| `workspace_not_allowed` | 修正客户端绑定，不循环重试切换工作区 |
| `path_not_found` | 用 `list_revision_tree` 明确指定 rev 查询；磁盘存在不代表所选提交中存在 |
| 子模块的 `not_a_directory` / `not_a_file` | 连接子模块自己的仓库 |
| Git 指针损坏或目标不存在 | 修复原有 Git 元数据；失败初始化不会留下 `.jj` |
| 创建结果提示 HEAD 差异 | 核对历史基线是否有意选择，避免在旧实现上继续开发 |

优先复用客户端已配置的 MCP 连接。程序化接入使用正式 MCP SDK，保持会话至任务结束；设置请求超时，持续消费 stderr，并处理服务端请求、进度、取消及连接关闭。stdio stdout 只承载协议。

分别检查 JSON-RPC `error`、MCP `result.isError` 和业务结果。`run` 还需检查 `exit_code`、`timed_out`、`cancelled` 及 snapshot 结果。外层 Python/PowerShell 退出码为 0，不代表内部工具或构建命令成功。


MCP 统一通过 stdio 接入；编排者可使用 `--integrator` 并显式指定工作区。原生文件工具和终端不会随 MCP 绑定自动切换目录。任务开始保存固定 commit，结束时比较完整任务差异；检查 status 的截断标记。
