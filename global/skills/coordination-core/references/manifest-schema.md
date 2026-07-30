# Manifest Schema

`manifest.yaml` 由当前协调者创建和更新。顶层结构始终完整；契约尚未发布时，发布后才有值的字段按下述生命周期使用 `null`。更新时保留任务已消耗的 Workflow 预算语义。

```yaml
schema_version: 1
task_id: rename-character
task_state: ready
coordination_mode: manual-sessions
coordinator:
  type: human
  identifier: user
project_root: /Projects/MMORPG
contract:
  source: /Projects/MMORPG/docs/contracts/rename-character.md
  version: v001
  sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
workflow:
  mode: auto
  effective_budget: heavy
  budget_scope: task
  ledger: /Projects/MMORPG/.harness/tasks/rename-character/integration/workflow-events.yaml
  ledger_last_sequence: 0
  ledger_sha256: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
roles:
  client:
    workspace: /Projects/MMORPG-client
    rules:
      - /Projects/MMORPG/AGENTS.md
      - /harness/src/client/AGENTS.md
    writable:
      - /Projects/MMORPG-client/Src/Client
  server:
    workspace: /Projects/MMORPG-server
    rules:
      - /Projects/MMORPG/AGENTS.md
      - /harness/src/server/AGENTS.md
    writable:
      - /Projects/MMORPG-server/Src/Server
  test:
    workspace: /Projects/MMORPG-test
    rules:
      - /Projects/MMORPG/AGENTS.md
      - /harness/src/test/AGENTS.md
    writable:
      - /Projects/MMORPG-test/Src/Tests
updated_at: 2026-07-30T12:00:00+08:00
```

## 字段约束

- `schema_version`：必填整数，当前为 `1`。
- `task_id`：必填，满足任务标识正则；创建后不可变。
- `task_state`：必填，取值服从任务状态机。
- `coordination_mode`：必填，只能为 `manual-sessions` 或 `leader-subagents`。
- `coordinator.type`：必填，只能为 `human` 或 `primary-agent`；`identifier` 必填且能标识当前协调者。
- `project_root`：必填，已验证的规范化绝对路径。
- `contract.source`：从 `draft` 起必填正式契约绝对路径。`draft`、`contract-review` 阶段的 `version` 和 `sha256` 必须为 `null`；发布快照后，`version` 匹配 `^v[0-9]{3}$`，`sha256` 为对应快照的 64 位小写十六进制摘要。任务进入 `ready` 前两者必须已有非空合法值。
- `workflow.mode`：必填，值来自当前 Workflow；`effective_budget` 是当前任务的消费结果；`budget_scope` 固定为 `task`。`ledger` 是核心任务布局中的绝对事件账本路径，`ledger_last_sequence` 是协调者已确认的最后连续序号，初始化为 `0`；`ledger_sha256` 是账本当前精确文件字节的 64 位小写十六进制摘要。
- `roles`：`client`、`server`、`test` 均必填；每项的 `workspace`、`rules`、`writable` 使用绝对路径，规则和可写路径不得靠聊天补充。
- `updated_at`：必填 RFC 3339 时间，由协调者在每次 Manifest 修改时更新。

`effective_budget` 不在协调模式切换时重置。数值配额不写入 Manifest；预算解释与累计消费只读取当前 Workflow Skill 和 `global/rules/workflow-policy.md`。

## Workflow 消费账本

`integration/workflow-events.yaml` 是单任务唯一的 Leader 模式追加式消费记录。只有 `coordination_mode: leader-subagents` 时由 Leader 写入；已有事件不可删除、改序或改写。手动模式的 `integration`、`client`、`server`、`test` 会话分别遵循各自当前 Workflow，不写此账本，也不跨会话聚合剩余预算。文件初始化为 `events: []`，事件结构为：

```yaml
schema_version: 1
task_id: rename-character
events:
  - sequence: 1
    occurred_at: 2026-07-30T12:20:00+08:00
    event: agent-dispatched
    category: read-only-agent
    executor_id: inspect-contract-01
    source_binding: integration-leader-01
    outcome: pending
    counts_toward_budget: true
```

- `sequence` 从 `1` 连续递增；`occurred_at` 为 RFC 3339 时间。
- `event` 只能为 `agent-dispatched`、`agent-outcome`、`verification-started` 或 `clarification-started`。
- `category` 只能为 `read-only-agent`、`read-write-agent`、`test-round` 或 `clarification-round`，并与当前 Workflow 策略的计数类别一致。
- `executor_id` 标识 Agent、验证目标或 Checkpoint；`source_binding` 标识记录事件的协调绑定。
- `outcome` 只能为 `pending`、`completed`、`failed`、`interrupted` 或 `rework`。派发、验证或澄清开始事件立即计数；后续结果用新的 `agent-outcome` 事件记录，不改写原事件，也不重复计数。
- `counts_toward_budget` 必填布尔值。是否计数服从 `global/rules/workflow-policy.md`；账本不定义数值配额。

Leader 模式每次派发、开始验证或进入普通澄清前，先追加计数事件，计算整个账本精确文件字节的 SHA-256，再更新 Manifest 的 `ledger_last_sequence` 和 `ledger_sha256`。Leader 恢复时校验 task_id、连续序号、Manifest 指针和摘要，从全部 `counts_toward_budget: true` 的开始事件重算累计消费；失败、中断和返工只改变结果，不撤销原计数。账本缺失、序号断裂、指针不一致或摘要漂移时停止派发并报告状态损坏，不按聊天历史补数。

从手动模式切换到 Leader 时保留已有账本，不计入其他手动会话的独立消费；由当前主 Agent 从切换完成后的首次 Leader 行为开始追加。切换回手动模式后停止追加，但不删除或重置历史 Leader 事件；以后恢复 Leader 模式时继续累计。账本与 Manifest 都由协调者维护；摘要用于发现意外漂移，不构成抵抗协调者同时恶意篡改两处的安全边界。
