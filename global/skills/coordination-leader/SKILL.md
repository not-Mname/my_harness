---
name: coordination-leader
description: Use when the current primary Agent must initialize or resume a persistent cross-role task, dispatch role subagents, enforce workflow budgets, and integrate their handoffs.
---

# Leader 子代理协调

## 必需协议

**必需子技能：** 使用 `coordination-core`。初始化、恢复、派发和汇合前，完整读取其入口及当前操作要求的引用；本 Skill 不复制共享 Schema、状态机或所有权。

当前主 Agent 是 Leader 和协调者。角色子 Agent 不自行集成、不修改共享契约，也不继续派发。

## 命令

```text
$coordination-leader [action=<init|resume>] \
  root=<absolute-path> [task=<task-id>]
```

- `root` 始终必填；缺失或非法时主动询问有效的绝对项目路径，不从当前目录或旧绑定继承。
- 提供 `task` 且省略 `action` 时默认 `init`；未提供 `task` 时默认 `resume`。
- `init` 必须显式提供合法 `task`；缺失时主动询问 `task=<task-id>`。
- `resume` 可显式提供 `task`；省略时只读取 `<root>/.harness/current-task`，指针缺失、非法或悬空时主动询问 `task=<task-id>`，不得从聊天、分支、目录或时间猜测。
- `action` 非 `init` 或 `resume` 时主动询问合法 action；action 缺失时按上面的 task 规则决定默认 resume 或 init。

缺参询问示例：

```text
为了恢复 Leader 任务，还需要：root（已存在的绝对项目路径）。
未提供 action，将按 resume 处理；如果不使用 current-task，请同时提供 task=<task-id>。
```

`init` 按 `coordination-core` 创建任务骨架、正式契约、不可变快照和 Manifest。目标项目未忽略 `.harness/` 或正式契约需要确认时，先发布 HITL；契约未冻结不得把任务置为 `ready` 或派发角色。

`resume` 不重新初始化。依次读取 Manifest、正式契约、当前快照、全部角色状态、未决 blocker、已有 handoff、集成状态和 Manifest 指向的 Workflow 消费账本；按核心 Schema 校验并重算累计消费。

## 模式与所有权门禁

- Leader 切换不能修改 Workflow 模式或重置已消耗预算。
- 只有所有角色均为 `unassigned | released | handoff-ready | completed` 时才能从手动模式切换。
- 任一角色存在 `manual-session + active`、`blocked` 或 `contract-changed` 绑定时，停止整任务接管并发布 HITL，请用户等待、完成或授权释放。
- 不抢占活跃手动角色，也不先派发其余角色形成混合模式。
- 存在活跃 `leader-subagent` 时按恢复协议等待或汇合，不创建第二绑定。

## Workflow 预算

1. 开始任务时读取当前 Workflow Skill 和 `global/rules/workflow-policy.md`。
2. Workflow 为 `auto` 时，在当前任务开始时确定有效预算；固定模式直接消费其预算。
3. 每次派发前先按 `coordination-core` 追加 Workflow 计数事件；已完成、失败、中断和返工 Agent 均计入本任务累计派发，结果事件不撤销原计数；恢复和协调模式切换不清零。
4. 派发时冻结 Agent 的只读或可读写身份；只读 Agent 不得升级为可读写 Agent。
5. 生成或更新计划、失败分组，以及每次派发前，重新构建依赖层并检查剩余预算和平台并发。
6. 本 Skill 不复制数值配额、不创建预算例外；用户显式 Skill 覆盖、自动升级和固定模式残余风险完全服从 `workflow-policy.md`。
7. 验证门禁与剩余预算冲突时按 Workflow 策略处理并报告，不能静默省略验证。

## 派发决策

先冻结正式契约、版本和哈希，再划分路径所有权与依赖层。只有输入确定、写入互斥且无共享可变状态的角色才能同层并行。典型依赖层为：

```text
冻结契约 → client + server → test → Leader 串行集成与最终验证
```

若 test 不依赖实现产物，或 client/server 存在额外依赖，按实际契约重建依赖层，不机械套用示例。

每个角色使用完整、自包含的提示词；禁止使用“上文”“同前”或隐含聊天历史：

```text
[ROLE-TASK]
task_id: <合法 task-id>
role: client | server | test
workspace: <绝对路径>
rules: <逐项绝对路径>
contract_source: <绝对路径>
contract_snapshot: <绝对路径>
contract_version: vNNN
dependency_summary: <冻结的依赖与接口摘要>
writable_paths: <绝对路径列表>
read_only_paths: <绝对路径列表>
acceptance: <可验证条件>
verification: <实际命令或明确的发现步骤>
handoff_path: <绝对路径>
forbidden: 不扩大范围、不修改契约、不自行集成、不继续派发
```

派发前逐项确认 `workspace`、规则、契约源、快照、版本、可写路径与 handoff 路径都来自当前 Manifest，并在角色状态中创建唯一 `binding_id` 和 `executor_mode: leader-subagent`。契约版本在派发写入前后变化时进入 `contract-changed`，取消该次派发并重新评估。

## Blocker 与汇合

角色按 `coordination-core` 写自身 blocker 和 handoff。涉及契约、数据、安全、路径所有权、外部副作用或项目设置的 blocker，由 Leader 汇总并通过 HITL 请求用户决策；角色不得自行补定。

Leader 按依赖顺序串行汇合：

1. 读取当前版本、Manifest、角色状态、blocker 和 handoff，不信任子 Agent 的完成声明。
2. 比较实际修改路径、角色授权路径和 `changed_files`；越界时拒绝集成。
3. 检查所有角色使用相同的正式契约版本和哈希；子 Agent 擅改正式契约或快照时拒绝集成。
4. 核对实际验证命令、退出码、结果、风险和剩余工作；按依赖顺序处理 handoff，不让角色自行集成。
5. 在剩余验证预算内运行最小充分的最终验证；无法兼顾时按 Workflow 策略处理并显式报告。
6. 只有实际文件与新鲜验证证据支持时，才能把角色和任务标记为 `completed`。
