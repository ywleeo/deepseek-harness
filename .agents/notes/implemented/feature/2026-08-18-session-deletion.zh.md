# Agent Note：会话删除（workspace.deleteSession）

状态：已实现

[English](2026-08-18-session-deletion.md) | 中文

## 问题

会话行只提供**归档**（一种仅隐藏的显示层操作，保留日志与记账席位），而原本纯视觉的「删除会话」占位在归档实现时被直接改成了归档、没有处理函数。归档时的产品决策是「归档而非删除」；用户的真实需求是永久移除一次性问答会话：持久化日志必须消失（回收磁盘）、workspace 记账席位必须消失、归档集合成员必须消失，所有客户端视图（workspace 分组、Ungrouped、搜索、单列表）都必须删除该行。

## 决策

**为 seam 增加 `SessionPersistence.delete(id)`，为 workspace 注册表增加清理记账与归档集合的 `deleteSession`，并新增一元 `workspace.deleteSession`，由它编排日志删除 + 记账清理，并为冷会话发出 `host/session-removed` 帧。**

- 持久化 seam：`SessionPersistence.delete(id)` 为抽象方法。coordinator 以按 id 串行化的操作实现它：等待退役完成、失效 prepared 源、调用新的可选后端钩子 `deleteStored(id)`、移除内存状态。没有 `deleteStored` 的后端会收到明确错误；两个一方后端都实现了它——JSONL 移除会话的 `<root>/<project>/<session-id>/` 目录，SQLite 删除 `sessions` 行（事件级联）。seam 的 README Known Limitations 从「无删除或保留 API」改为「保留策略仍延后；删除已存在」。
- Workspace 注册表：`deleteSession(sessionId)` 走注册表操作链。它校验会话已知（实时或已持久化）——未知 id 抛 `WorkspaceUnknownSessionError`——并拒绝**实时**会话（新 `WorkspaceSessionLiveError`：宿主会丢弃 `AgentHandle`，当前无法处置实时 agent，在活跃 write-behind 之下删除会损坏日志），随后从每个 workspace 的 `sessionIds` 记账和 `archivedSessionIds` 中移除该 id，持久化完成。注册表在持久写入后发出 cordis 事件 `session/deleted`（声明在会话域的 Events 上，与 `session/disposed` 并列）。
- RPC：`workspace.deleteSession({sessionId}) → { deleted: true }`。未知 id 映射到既有 `session-not-found` 错误码；实时会话映射到新 `session-live` 码。handler 先调用 `sessionPersistence.delete(id)` 再调用 `workspaceRegistry.deleteSession(id)`；宿主流的 `session/deleted` 监听器推送 `host/session-removed`，复用客户端已有的完整移除处理（summary 移除、会话 removed 标记、选择回落到 New Session 视图）。message-feedback 服务在同一事件上级联：其按 Session 的伴随行在同一操作尾上被删除，因此产品内删除路径不会留下孤儿反馈行。
- 客户端运行时：`IWorkspaces.deleteSession` + `WorkspaceRuntime.deleteSession` + `WorkspaceManager.deleteSession` 调用该一元方法；行通过既有的 `host/session-removed` 路径消失，因此客户端列表无需额外手术。
- UI：会话行菜单新增**删除会话**项（危险样式、确认对话框——归档项保持非破坏、无对话框）。中英文案。

## 备选方案

**由宿主带外删除日志文件（功能前的「维护」答案）。** 否决：绕过 seam，SQLite 后端（无按会话产物）无法工作，且会留下过期的 workspace 记账与归档集合。

**在 RPC 拒绝实时会话、让客户端先处置。** 本次迭代否决：`AgentRegistry` 没有按 id 处置能力，宿主丢弃了每一个 `AgentHandle`（见 per-session-agent-presets note），因此实时处置路径是独立的生命周期功能。`session-live` 是今天诚实的边界；错误文本会告知用户该会话当前处于打开状态。

**冷移除复用 `session/disposed`。** 否决：它只对实时会话触发；新事件 `session/deleted` 是冷会话的对应物，映射到同一个 `host/session-removed` 帧。

## 后果

- 删除当前打开的会话会被拒绝（`session-live`）；客户端展示宿主消息。在处置能力落地前，删除「agent 已附加但空闲」的会话同样被拒绝。
- 删除是终局性的：无回收站、无取消归档。日志与记账在一个持久序列中移除；两者之间崩溃会留下日志已删除、记账席位过期的情况——启动时的 header 索引过滤本就容忍它（报告为「session header is missing」），后续注册表变更会剪除。
- `workspace.list` 响应形状不变（被删会话只是离开 `sessionIds`）；归档集合经既有全快照姿态收缩。
- workspace-management e2e 固定归档往返（既有）；删除流程由领域测试覆盖：未知拒绝、实时拒绝、记账/归档集合清理、重启恢复。持久化测试固定 coordinator delete（退役等待、prepared 源失效、缺钩子拒绝）与两个后端 `deleteStored` 实现。反馈级联测试固定 `session/deleted` 时行被移除。
