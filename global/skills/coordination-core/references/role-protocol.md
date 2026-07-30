# 角色、Blocker 与 Decision 协议

## 读写矩阵

| 身份 | 可写 | 只读 |
|---|---|---|
| `integration` / 协调者 | 正式契约、Manifest、快照、assignment、`integration/**` | 全部角色状态、blocker、handoff 和业务产物 |
| `client` | Manifest 授权的客户端路径及 `roles/client/{status.yaml,handoff.yaml,blockers/**}` | 正式契约、当前快照、Manifest、assignment、其他角色产物 |
| `server` | Manifest 授权的服务端路径及 `roles/server/{status.yaml,handoff.yaml,blockers/**}` | 同上 |
| `test` | Manifest 授权的测试路径及 `roles/test/{status.yaml,handoff.yaml,blockers/**}` | 同上 |

角色不能直接改写彼此状态，不能修改正式契约或通过其他角色实现补全契约。

## 绑定

角色进入 `active` 时创建当前执行者唯一且稳定的 `binding_id`。`executor_mode` 只能为 `manual-session` 或 `leader-subagent`。

```yaml
task_id: rename-character
role: client
executor_mode: manual-session
binding_id: client-session-01
state: active
contract_version: v001
updated_at: 2026-07-30T12:00:00+08:00
```

- 绑定前读取角色状态；已有活跃绑定时拒绝第二个执行者。
- Leader 不抢占活跃 `manual-session`，手动会话不抢占活跃 `leader-subagent`。
- 只有原执行者主动释放或用户明确授权释放，状态才能进入 `released`。
- 恢复者复用任务状态但创建自己的新 `binding_id`；必须重新读取 Manifest、正式契约、当前快照、角色状态、未决 blocker 和旧 handoff。
- 绑定、释放或恢复均须在写入前后核对 `contracts/current`。

初始化时，协调者为每个业务角色创建尚未绑定的状态：

```yaml
task_id: rename-character
role: client
executor_mode: null
binding_id: null
state: unassigned
contract_version: null
updated_at: 2026-07-30T12:00:00+08:00
```

契约达到 `ready` 后，协调者为每个角色创建 `assignment.md`。它至少包含 `task_id`、`role`、绝对 `workspace`、逐项绝对规则路径、正式契约与快照绝对路径、契约版本和哈希、依赖摘要、可写与只读绝对路径、验收条件、验证命令或发现步骤，以及 handoff 绝对路径。assignment 不保存活跃 `binding_id`，角色执行者不得修改它。

`integration` 会话不占用业务角色，其绑定持久化在 `integration/status.yaml`，由当前协调者维护。初始化期间允许它在 `draft` 状态以 `contract_version: null` 活跃；正式契约发布后，协调者在同一绑定上更新为当前版本，再将任务置为 `ready`：

```yaml
schema_version: 1
task_id: rename-character
role: integration
executor_mode: manual-session
binding_id: integration-session-01
state: active
contract_version: v001
updated_at: 2026-07-30T12:00:00+08:00
```

Leader 模式将 `executor_mode` 写为 `leader-subagent`，`binding_id` 标识当前主 Agent。`integration/status.yaml` 不授予任何业务角色写入权，也不能绕过业务角色的独占绑定。

## Blocker

文件名为 `roles/<role>/blockers/<role>-<sequence>.yaml`。序号单调递增；文件不删除，`status` 只能为 `open`、`resolved` 或 `rejected`。

```yaml
schema_version: 1
id: client-001
task_id: rename-character
role: client
type: contract-gap
contract_version: v001
summary: 缺少超时后查询最终结果的定义
impact: 客户端无法实现可靠重试
requested_decision: 明确查询接口和 requestId 幂等语义
status: open
resolution_ref: null
created_at: 2026-07-30T12:00:00+08:00
```

遇到契约缺口、路径越权、数据、安全、外部副作用或项目设置问题时，角色写 blocker 并停止受影响工作。

## Decision

文件名为 `integration/decisions/decision-<sequence>.yaml`，只由协调者创建。涉及强制确认的决策必须记录对应任务内唯一的 HITL Checkpoint。

```yaml
schema_version: 1
id: decision-001
task_id: rename-character
resolves:
  - roles/client/blockers/client-001.yaml
checkpoint: CP-3
summary: 使用原 requestId 查询最终结果
contract_version_before: v001
contract_version_after: v002
status: accepted
created_at: 2026-07-30T12:10:00+08:00
```

解决 blocker 时先写 decision 或记录拒绝依据，再更新 blocker 的 `status` 和 `resolution_ref`；不覆盖历史内容。
