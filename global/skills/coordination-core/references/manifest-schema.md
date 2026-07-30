# Manifest Schema

`manifest.yaml` 由当前协调者创建和更新。所有字段必填；更新时保留任务已消耗的 Workflow 预算语义。

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
- `contract.source`：必填正式契约绝对路径；`version` 匹配 `^v[0-9]{3}$`；`sha256` 为对应快照的 64 位小写十六进制摘要。
- `workflow.mode`：必填，值来自当前 Workflow；`effective_budget` 是当前任务的消费结果；`budget_scope` 固定为 `task`。
- `roles`：`client`、`server`、`test` 均必填；每项的 `workspace`、`rules`、`writable` 使用绝对路径，规则和可写路径不得靠聊天补充。
- `updated_at`：必填 RFC 3339 时间，由协调者在每次 Manifest 修改时更新。

`effective_budget` 不在协调模式切换时重置。数值配额不写入 Manifest；预算解释与累计消费只读取当前 Workflow Skill 和 `global/rules/workflow-policy.md`。
