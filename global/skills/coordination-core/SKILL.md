---
name: coordination-core
description: Use when manual Codex sessions or a leader Agent need a shared cross-role task protocol, including persistent contracts, role ownership, blockers, handoffs, recovery, or coordination-mode transitions.
---

# 跨角色协调协议

## 核心边界

本 Skill 是协调协议的唯一事实源，不切换协调模式，不自行派发 Agent。
正式契约位于 `docs/contracts/`；`.harness/` 只保存本机运行状态和不可编辑快照。

## 必读引用

按当前操作完整读取：

- 初始化、路径或文件权限：[references/task-layout.md](references/task-layout.md)
- Manifest 或契约发布：[references/manifest-schema.md](references/manifest-schema.md)
- 角色绑定、blocker 或 decision：[references/role-protocol.md](references/role-protocol.md)
- Handoff 或完成验收：[references/handoff-schema.md](references/handoff-schema.md)
- 状态转换、恢复或模式切换：[references/status-machine.md](references/status-machine.md)

## 不可绕过门禁

- 不从聊天历史、分支、目录时间或当前工作目录猜测任务。
- 不允许两个执行者同时拥有同一角色。
- 不允许角色修改正式契约、其他角色目录或集成目录。
- 每次写入前后核对当前契约版本。
- Workflow 预算由当前模式和 `global/rules/workflow-policy.md` 定义，本 Skill 不复制配额。
- `.harness/` 未被目标项目忽略时，按 HITL 请求授权，不自行修改 `.gitignore`。

## 参数解析门

- 发现必填参数缺失、参数值非法或任务指针无法解析时，主动向用户询问，不直接结束，也不从旧会话、聊天历史、当前目录、分支名或时间推断。
- 同一轮合并询问所有已发现的缺项，并列出允许值、当前已知值和缺失值。
- 参数得到补充后重新执行完整解析和路径/所有权校验；用户未补充前不创建目录、不绑定角色、不派发 Agent。
