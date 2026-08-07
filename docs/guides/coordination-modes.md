# 跨角色协调模式使用指南

## 模式选择

跨客户端、服务端和测试的任务支持两种协调方式：

| 模式 | 入口 | 协调者 | 适用场景 |
|---|---|---|---|
| 人工多会话 | `$coordination-manual` | 用户 | 希望亲自创建、分配和管理多个独立会话 |
| Leader 子代理 | `$coordination-leader` | 当前主 Agent | 希望由一个主 Agent 自动拆分、派发和汇总 |

两种模式共用 `$coordination-core` 定义的契约、任务状态、角色所有权、blocker 和 handoff 协议。通常不需要直接调用 `$coordination-core`。

同一任务同一时刻只能使用一种协调模式，不能让人工会话和 Leader 子代理分别占用不同角色。

## 缺少参数时

入口不会因为缺参直接结束，也不会猜测参数。它会合并当前已发现的缺项，一次询问允许值：

```text
为了加入协调任务，还需要：role（integration/client/server/test）、root（已存在的绝对项目路径）。
当前 action 未提供，将按 join 处理；请补充缺失参数后继续。
```

`task` 在普通 `join` 或 `resume` 中可省略；系统会读取 `<root>/.harness/current-task`。只有指针缺失、非法或悬空时，才询问显式 `task=<task-id>`。`init` 始终需要显式任务标识。

## 前置准备

命令中的 `root` 必须是真实业务项目的绝对路径，不能填写保存 harness 配置的目录。

目标项目建议具备以下结构：

```text
/Projects/MMORPG/
├── AGENTS.md
├── docs/
│   └── contracts/
└── .gitignore
```

`.harness/` 保存本机跨会话运行状态，不应提交到 Git。建议在目标项目的 `.gitignore` 中加入：

```gitignore
.harness/
```

如果目标项目尚未忽略 `.harness/`，协调 Skill 会发布 HITL Checkpoint 请求授权，不会自行修改 `.gitignore`。

## 人工多会话模式

人工模式中，用户是唯一 coordinator。每个 Codex 会话只承担一个明确角色，不派发子 Agent，也不替其他角色集成。

### 初始化任务

先打开一个 integration 会话：

```text
$coordination-manual role=integration action=init \
  root=/Projects/MMORPG task=rename-character
```

初始化流程包括：

1. 校验项目根目录和 `task-id`。
2. 创建 `.harness/tasks/rename-character/` 任务状态目录。
3. 创建正式契约 `docs/contracts/rename-character.md`。
4. 等待用户确认正式契约。
5. 生成不可变契约快照、版本和 SHA-256 摘要。
6. 创建 client、server、test assignment。
7. 将任务置为 `ready`，并更新 `.harness/current-task`。

遇到契约确认 Checkpoint 时，按编号回复：

```text
c CP-1
```

### 加入客户端角色

在新的 Codex 会话中执行：

```text
$coordination-manual role=client root=/Projects/MMORPG
```

任务参数可以省略。Skill 会读取：

```text
/Projects/MMORPG/.harness/current-task
```

也可以显式指定任务：

```text
$coordination-manual role=client \
  root=/Projects/MMORPG task=rename-character
```

### 加入服务端和测试角色

分别在新的会话中执行：

```text
$coordination-manual role=server root=/Projects/MMORPG
```

```text
$coordination-manual role=test root=/Projects/MMORPG
```

各角色默认职责如下：

| 角色 | 职责 |
|---|---|
| `integration` | 维护契约、assignment、decision、集成状态和最终验收 |
| `client` | 客户端实现，只写 Manifest 授权的客户端路径 |
| `server` | 服务端实现，只写 Manifest 授权的服务端路径 |
| `test` | 契约、集成和端到端测试，只写 Manifest 授权的测试路径 |

同一角色已有活跃绑定时，新会话不能抢占或自行释放旧绑定。

### 恢复 integration 会话

原 integration 会话结束或已合法释放后，可以在新会话中恢复：

```text
$coordination-manual role=integration action=join \
  root=/Projects/MMORPG
```

恢复过程会重新读取 Manifest、正式契约、当前快照、全部角色状态、未决 blocker 和已有 handoff，不依赖旧聊天记录。

### 处理 blocker

角色遇到契约缺口、越界需求、数据或安全决策、外部副作用或项目设置变更时，会停止受影响工作并写入 blocker：

```text
.harness/tasks/rename-character/roles/client/blockers/client-001.yaml
```

回到 integration 会话处理：

```text
检查并处理 rename-character 的未决 blocker
```

如果需要修改正式契约，integration 会先通过 HITL 请求确认，然后发布新的不可变快照，例如 `v002`。检测到契约版本变化的角色会进入 `contract-changed`，停止相关写入并重新读取契约。

### 汇总角色交付

角色完成实现和验证后，会写入自己的 `handoff.yaml` 并进入 `handoff-ready`。

回到 integration 会话执行：

```text
检查 client、server、test 的 handoff，执行最终集成和验收
```

只有用户协调者核对实际变更路径、契约版本和验证证据后，才能将角色和任务标记为 `completed`。

## Leader 子代理模式

Leader 模式只需要一个主会话。当前主 Agent 负责契约、依赖拆分、子 Agent 派发、handoff 汇总和最终验证。

### 初始化并执行任务

```text
$coordination-leader action=init \
  root=/Projects/MMORPG task=rename-character
```

Leader 会先初始化任务并冻结正式契约。契约达到 `ready` 后，典型依赖层为：

```text
冻结契约 → client + server → test → Leader 集成与最终验证
```

这只是典型结构。只有依赖已经确定、写入路径互斥且没有共享可变状态的角色才会并行。

每个子 Agent 收到的任务都会显式包含：

- 任务和角色。
- 绝对工作区。
- 需要读取的规则文件。
- 正式契约、快照和版本。
- 依赖与接口摘要。
- 可写和只读路径。
- 验收条件与验证方式。
- handoff 路径。
- 禁止扩大范围、修改契约、自行集成和继续派发的约束。

### 恢复已有任务

新开主会话或任务中断后执行：

```text
$coordination-leader action=resume root=/Projects/MMORPG
```

也可以显式指定任务：

```text
$coordination-leader action=resume \
  root=/Projects/MMORPG task=rename-character
```

Leader 会重新读取 Manifest、契约快照、角色状态、blocker、handoff 和 Workflow 消费账本。已完成、失败、中断和返工 Agent 都保留在累计消费中，不会因恢复而清零。

## 模式切换

### 人工模式切换到 Leader

所有人工角色必须处于以下状态之一：

```text
unassigned | released | handoff-ready | completed
```

然后执行：

```text
$coordination-leader action=resume root=/Projects/MMORPG
```

如果仍有 `manual-session` 角色处于 `active`、`blocked` 或 `contract-changed`，Leader 会停止接管并请求用户决定等待、完成或授权释放。Leader 不会抢占该角色，也不会先派发其他角色形成混合模式。

### Leader 模式切换到人工模式

先让所有 Leader 子 Agent 完成或释放，再在 integration 会话执行：

```text
$coordination-manual role=integration action=join \
  root=/Projects/MMORPG
```

模式切换完成后，再分别打开 client、server 和 test 人工会话。普通角色不能自行触发模式切换。

模式切换会保留 Manifest、契约快照、blocker、handoff 和历史 Leader Workflow 账本，不重新初始化任务。

## 任务标识规则

`task-id` 只能包含小写字母、数字和连字符，最长 64 个字符：

```text
rename-character
payment-refund
user-profile-v2
```

以下值非法：

```text
RenameCharacter
../rename-character
client/rename
rename character
```

## 命令速查

```text
# 人工模式：初始化
$coordination-manual role=integration action=init root=<项目绝对路径> task=<任务ID>

# 人工模式：恢复 integration
$coordination-manual role=integration action=join root=<项目绝对路径>

# 人工模式：加入业务角色
$coordination-manual role=client root=<项目绝对路径>
$coordination-manual role=server root=<项目绝对路径>
$coordination-manual role=test root=<项目绝对路径>

# Leader 模式：初始化
$coordination-leader action=init root=<项目绝对路径> task=<任务ID>

# Leader 模式：恢复
$coordination-leader action=resume root=<项目绝对路径>
```

## 使用建议

- 希望亲自控制每个会话、逐个查看结果时，使用 `$coordination-manual`。
- 希望一个主 Agent 自动拆分、派发和汇总时，使用 `$coordination-leader`。
- 任务契约或角色状态不明确时，先回到 integration 或 Leader 会话处理，不让业务角色自行补定。
- 不通过聊天内容、分支名、目录时间或当前工作目录猜测任务；显式提供 `task`，或使用 `.harness/current-task`。
