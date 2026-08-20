# Agent Note：会话关闭（实时 agent 处置）与空闲退役

状态：已实现

[English](2026-08-18-session-close.md) | 中文

## 问题

`workspace.deleteSession`（见[会话删除特性](2026-08-18-session-deletion.zh.md)）对每个**实时**会话都返回 `session-live` 拒绝，且错误文案要求用户“先关闭会话”——但当时根本不存在关闭能力。宿主丢弃了每一个 `AgentHandle`，`AgentRegistry` 也没有 dispose-by-id，因此任何曾被打开或发过消息的会话都会一直保持附加状态（其 agent 常驻内存），直到宿主进程退出：删除“未打开”的会话会失败，而实时集合随每个打开的会话无界增长。

## 决策

**为 `AgentRegistry` 增加 dispose-by-id，新增 `session.close` 一元方法，`workspace.deleteSession` 改为先关闭再删除，为关闭路径新增 `host/session-closed` 帧，并加入可配置的空闲退役扫描以处置停稳 agent。**

- `AgentRegistry.create`/`resume` 现在把返回句柄的 disposer 留存进按 id 索引的映射（在 `agent/disposed` 时移除），新的 `dispose(id, guard?)` 会执行该 disposer——即 loop 的完整 teardown（停止、等待完全停稳使 write-behind 排空、注销 agent、detach 其会话、展开 scope）。可选的 `guard` 在 disposer 触发 abort 之前同步求值，因此重新检查停稳状态的调用方（如空闲退役）不会把一次刚到达的 prompt 竞态成取消。
- `session.close({sessionId})`——处置实时 agent；对已冷的持久化会话返回幂等的 `{closed: true}`；明确未命中返回 `session-not-found`；会话型 subagent 返回 `agent-busy`（所有权 fence 与其它动词一致）。
- `workspace.deleteSession` 是先关闭再删除：网关在调用注册表之前先处置实时 agent（含 subagent fence），因此已附加但空闲的会话像冷会话一样被删除。`session-live` 只在一个会话于关闭→删除的竞态窗口内被重新恢复时出现，注册表仍拒绝实时会话作为非 web 调用方的安全网。
- **宿主流信号**：宿主流把 `session/disposed` 映射为 `host/session-removed`（对 subagent detach 是承重的）。关闭会把 id 记入网关本地 `closingSessions` 集合（持续到处置结束），因此客户端改为收到新增的 `host/session-closed` 帧：行**保留**（关闭的会话是冷的，不是被删除）、running 翻转为 false、仅实时存在的状态（jobs、待处理缓冲/交互）被丢弃——镜像移除路径但不做移除。删除仍发出 `host/session-removed`（dispose 与 `session/deleted` 各一次）。
- **空闲退役**（网关配置 `idleSessionCloseMs`，默认 30 分钟，`0` 关闭）：每 60 秒扫描一次，处置超过阈值一直保持停稳（idle 状态、空 inbox）的 agent。时钟从 agent 首次被观察到停稳开始；任何活动（状态翻转或 inbox 出现工作）都会重置。退役向 `dispose` 传入停稳 guard，且绝不针对 subagent 拥有的身份。被退役的会话只是变冷——下一次 prompt 会透明地恢复它。
- 客户端运行时：`ISessions.close` + `SessionRuntime.close` + `SessionManager.close` 调用该一元方法；`SessionManager.handleHostEnvelope` 新增 `host/session-closed` 分支。UI：会话行菜单新增非破坏性的**关闭会话**项（无对话框，与归档一致）；中英文案。

## 备选方案

**让注册表直接处置实时会话来实现删除。** 拒绝：agent 的 teardown 属于 agent 层（`packages/core/agent`），注册表（`packages/workspace`）不应拥有它；网关在具备完整上下文（subagent fence、持久化）的地方编排先关闭再删除。

**复用 `host/session-status { running: false }` 表示关闭。** 拒绝：客户端无法区分“agent 转入空闲”（jobs 保留）与“agent 已被处置”（jobs/待处理状态消亡），因此关闭需要自己的帧，并在移除路径的清理基础上减去移除动作。

**事件驱动的空闲检测（在 idle 信号上退役）。** 拒绝，改为有界的周期扫描：带同步停稳 guard 的扫描比把所有唤醒/结算边界接成事件更易推理和测试；扫描节奏（60 秒）远小于默认阈值。

## 后果

- 误导性的 `session-live`/“（在客户端打开）”体验消失：删除任意普通会话都能成功（先关闭），空闲会话自动关闭。
- 实时集合的内存现在跟随近期活动，而不是跟随每个曾打开的会话。退役后的会话在冷态下零开销，下一次 prompt 透明恢复。
- `session.close` 与空闲退役会取消进行中的轮次（如果有）——关闭的语义就是 dispose；停稳 guard 只保护“在调用方自身检查之后才开始”的轮次。客户端保留该行，仍可继续输入——下一次 prompt 会冷恢复。
- Subagent 会话自始至终保持 fence：关闭、删除、退役都会拒绝它们（`agent-busy`／跳过），因为其 teardown 属于父 agent。
- 测试覆盖：`AgentRegistry.dispose`（guard 接受/拒绝、外部 teardown 后的 disposer 清理）、`session.close`（实时处置、冷幂等、未知、subagent fence、`host/session-closed` 帧）、删除-关闭（实时删除、subagent fence），以及纯函数 `pickIdleRetirementTargets` 的判定（停稳、截止时间、inbox、subagent 跳过、时钟启动）。
