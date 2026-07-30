# Handoff Schema

角色将交付写入自己的 `handoff.yaml`：

```yaml
task_id: rename-character
role: client
binding_id: client-session-01
contract_version: v001
status: handoff-ready

changed_files:
  - Src/Client/Example.cs

contract:
  consumed: v001
  proposed_changes: []

decisions:
  - summary: 复用现有请求状态组件
    evidence: Src/Client/Example.cs

verification:
  - command: dotnet test ...
    exit_code: 0
    result: passed

blockers_resolved: []
risks: []
remaining_work: []
generated_at: 2026-07-30T12:00:00+08:00
```

## 交付门禁

- `status` 必须为 `handoff-ready`，且不存在 `open` blocker。
- `binding_id` 必须等于当前角色绑定。
- `changed_files` 全部位于该角色的 Manifest 授权路径；使用相对项目根路径记录。
- `contract_version`、`contract.consumed` 必须等于 `contracts/current`，且摘要与 Manifest 一致。
- `verification` 每项都包含实际 `command`、整数 `exit_code` 和 `result`；未执行的验证不得伪造成成功，写入 `remaining_work`。
- `blockers_resolved`、`risks`、`remaining_work` 即使为空也保留数组。
- 角色不能在 handoff 中批准自己的契约提案或集成其他角色产物。
- 协调者必须复核实际变更、路径边界、契约版本和验证证据，才能把角色从 `handoff-ready` 标记为 `completed`。
