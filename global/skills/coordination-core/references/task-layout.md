# 任务布局与文件权限

## 路径校验

- `root` 必须是已存在、规范化后的绝对路径。
- `task-id` 必须匹配 `^[a-z0-9][a-z0-9-]{0,63}$`；拒绝点路径、斜杠、反斜杠、空白和路径编码。
- 解析真实路径后，任务目录必须仍位于 `<root>/.harness/tasks/` 内；否则停止。
- 未显式提供任务时，只能读取 `<root>/.harness/current-task`；指针缺失、非法或悬空时停止，不按聊天、分支、目录或时间猜测。

## 固定布局

```text
<root>/docs/contracts/<task-id>.md
<root>/.harness/current-task
<root>/.harness/tasks/<task-id>/manifest.yaml
<root>/.harness/tasks/<task-id>/contracts/current
<root>/.harness/tasks/<task-id>/contracts/v001.snapshot.md
<root>/.harness/tasks/<task-id>/roles/{client,server,test}/assignment.md
<root>/.harness/tasks/<task-id>/roles/{client,server,test}/status.yaml
<root>/.harness/tasks/<task-id>/roles/{client,server,test}/handoff.yaml
<root>/.harness/tasks/<task-id>/roles/{client,server,test}/blockers/
<root>/.harness/tasks/<task-id>/integration/decisions/
<root>/.harness/tasks/<task-id>/integration/workflow-events.yaml
<root>/.harness/tasks/<task-id>/integration/status.yaml
<root>/.harness/tasks/<task-id>/integration/report.md
```

该布局表示任务生命周期内允许出现的完整文件集合，不表示初始化时创建全部叶子文件。初始化只创建目录、`manifest.yaml`、三个角色的 `status.yaml`、`integration/status.yaml`、包含 `schema_version: 1`、当前 `task_id` 和 `events: []` 的可解析 `workflow-events.yaml`，以及契约发布所需文件；契约达到 `ready` 后由协调者创建 assignment。角色首次进入 `handoff-ready` 时才创建 `handoff.yaml`，不得用空文件或伪造的 `handoff-ready` 占位。`blockers/` 与 `integration/decisions/` 可在初始化时创建空目录。

`docs/contracts/` 是正式契约唯一可编辑来源。发布后生成 `v001`、`v002` 递增的不可变快照；禁止原地修改旧快照。`contracts/current` 仅保存当前版本号。

## 写入所有权

| 路径 | 唯一写入者 |
|---|---|
| `docs/contracts/**` | 当前协调者，经所需 HITL 确认 |
| `.harness/current-task`、`manifest.yaml`、`contracts/**` | 当前协调者 |
| `roles/<role>/assignment.md` | 当前协调者 |
| `roles/<role>/status.yaml` | 对应角色执行者；协调者仅创建初始 `unassigned` 状态及在验收后确认 `completed` |
| `roles/<role>/handoff.yaml`、`blockers/**` | 对应角色执行者 |
| `integration/**` | 当前协调者 |

角色只能写 Manifest 授权的业务路径，以及自己的 `status.yaml`、`handoff.yaml` 和 `blockers/`。角色不得写正式契约、共享文件、其他角色目录或 `integration/`。协调者负责 assignment、共享状态和集成产物。

`.harness/` 是本机状态，不进入 Git。目标项目尚未忽略它时，发布 HITL Checkpoint 请求修改 `.gitignore`；未经授权不得修改项目设置或初始化任务。
