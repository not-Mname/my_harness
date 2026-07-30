# 状态机、恢复与模式切换

## 任务状态

```text
draft → contract-review → ready → active → integrating → completed
active → blocked → active
任意未完成状态 → cancelled
```

只有协调者能更新任务状态。契约未经确认、快照和 Manifest 未发布时不得进入 `ready`；角色未验收时不得进入 `integrating`；最终验证无实际证据时不得进入 `completed`。

未列出的转换全部非法。非法请求不得部分写入状态，必须报告当前状态、请求转换和所缺前置条件。

## 角色状态

```text
unassigned → active → handoff-ready → completed
active → blocked → active
active → contract-changed → active
active → released → active
```

- 对应角色执行者可写 `active`、`blocked`、`contract-changed`、`released` 和 `handoff-ready`；协调者核对证据后写 `completed`。
- `blocked` 必须已有 blocker；恢复前必须已有 decision 或阻塞解除证据。
- `contract-changed` 表示当前绑定版本与 `contracts/current` 不同。执行者立即停止相关写入，重新读取正式契约、新快照、Manifest、角色状态、未决 blocker 和旧 handoff，重新评估并更新绑定版本后才能回到 `active`。
- `released` 只允许原执行者主动释放或用户明确授权释放；新执行者校验所有权并创建新绑定后才能回到 `active`。
- `handoff-ready` 必须满足 handoff 门禁；只有协调者能转为 `completed`。

## 恢复

新会话不继承聊天状态。恢复顺序固定为：解析显式任务或 `current-task`，读取 Manifest、正式契约、当前版本与快照、目标角色状态、未决 blocker、已有 handoff，再检查绑定和 Workflow 剩余预算。

协调模式切换不重置 Manifest、快照、blocker、handoff 或 Workflow 已消费预算，也不重新初始化任务。

## 模式切换

手动模式与 Leader 模式切换时，所有角色都必须处于：

```text
unassigned | released | handoff-ready | completed
```

存在 `active`、`blocked` 或 `contract-changed` 角色时停止切换，通过 HITL 由用户决定等待、完成或授权释放；不得抢占，也不得只接管其余角色形成混合执行。

切换为 Leader 时由当前主 Agent 成为协调者；切换为手动模式时用户成为协调者。第一版同一任务同一时刻只能使用一种执行模式。
