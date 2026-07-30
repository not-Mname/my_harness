---
name: coordination-manual
description: Use when a user coordinates independent Codex sessions that must initialize or join persistent integration, client, server, or test roles for one cross-role task.
---

# 人工多会话协调

## 必需协议

**必需子技能：** 使用 `coordination-core`。开始任何初始化、绑定、恢复或写入前，完整读取其入口及当前操作要求的引用文件；本 Skill 不重定义共享 Schema、状态机或所有权。

手动模式中用户是唯一协调者。当前 Agent 只承担显式指定角色，不成为 Leader，不派发子代理，也不替其他角色集成。

## 命令

```text
$coordination-manual role=<integration|client|server|test> \
  [action=<init|join>] root=<absolute-path> [task=<task-id>]
```

- `role`、`root` 始终必填；缺失时停止，不从旧会话绑定、聊天或当前目录继承。
- `action` 可选，默认 `join`。
- 合法组合：`integration + init`、`integration + join`、`client + join`、`server + join`、`test + join`。
- `client/server/test + init` 非法，立即拒绝且不创建目录。
- `init` 必须显式提供合法 `task`。
- `join` 的 `task` 可选；省略时只读取 `<root>/.harness/current-task`。指针缺失、非法或悬空时停止，不猜测任务。

调用成功后，模式、角色、项目、任务和 `binding_id` 在当前会话持续生效；新对话复位。再次显式调用本 Skill 才能重新绑定。切换角色、项目或任务时，先报告旧绑定与拟定新绑定，再按 `coordination-core` 校验所有权；不能沿用旧 `role` 或 `root` 补齐参数。

## `integration + init`

严格按以下顺序执行：

1. 校验参数、规范化 `root`、校验 `task-id`，并确认解析后的任务路径位于 `<root>/.harness/tasks/`。
2. 确认同名任务、任务目录和正式契约发布记录不存在；发现冲突时停止，不覆盖或转为隐式恢复。
3. 检查目标项目是否已忽略 `.harness/`。
4. 未忽略时发布任务内唯一的 HITL Checkpoint，请求授权修改 `.gitignore`；未获授权不修改项目设置、不创建任务。
5. 创建 `coordination-core` 定义的完整任务骨架，Manifest 状态为 `draft`，协调模式为 `manual-sessions`，协调者为用户。
6. 创建或读取 `<root>/docs/contracts/<task-id>.md`，将任务转为 `contract-review`；正式契约是唯一可编辑来源。
7. 契约经 HITL 明确确认后，生成递增的不可变快照、SHA-256 摘要和 `contracts/current`，并更新 Manifest。
8. 将任务置为 `ready`，最后更新 `<root>/.harness/current-task`。

契约未确认、快照或哈希未生成时，不得创建 `ready` 状态、角色 assignment 或活跃绑定。

## `join`

`integration` 或普通角色均按以下顺序加入：

1. 解析显式 `task`，否则读取并校验 `<root>/.harness/current-task`。
2. 读取 Manifest、正式契约、`contracts/current`、当前不可变快照，以及目标角色 assignment；`integration` 另读取集成状态。
3. 完整读取 Manifest `rules` 中列出的项目祖先链规则和显式角色 `AGENTS.md`；显式规则不因不在工作区祖先链而忽略。
4. 检查任务可实施状态、角色状态、现有 `binding_id` 和 `executor_mode`。已有 `manual-session` 或 `leader-subagent` 活跃绑定时拒绝抢占，返回用户协调；不得自行释放旧绑定。
5. 创建当前会话唯一 `binding_id`，将目标角色写为 `active`，`executor_mode` 写为 `manual-session`，并记录当前契约版本。
6. 状态写入前后读取 `contracts/current`；版本变化时进入 `contract-changed`，停止相关写入并按核心恢复协议重新评估。
7. 只在 Manifest 授权的角色业务路径和自身运行目录中工作；正式契约、共享状态、其他角色目录和 `integration/` 只读。

恢复不得依赖聊天历史。必须读取未决 blocker 和旧 handoff。遇到契约缺口、越界需求、数据或安全决策、外部副作用或项目设置变更时，在自身目录写 blocker，停止受影响工作并交回用户协调。

## 交付

提交 handoff 前再次核对 `contracts/current`，按 `coordination-core` 的统一 Schema 写入验证证据、风险和剩余工作。角色只能进入 `handoff-ready`；是否进入 `completed` 由用户核对实际变更和证据后决定。
