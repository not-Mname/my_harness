# 跨角色协调模式设计

## 目标

为跨客户端、服务端和测试的开发任务提供两种可显式切换的协调方式：

- 人工多会话模式：用户承担协调职责，分别创建并管理角色会话。
- Leader 子代理模式：当前主 Agent 承担协调职责并派发子代理。

两种模式必须复用同一任务协议、正式契约、运行状态、角色所有权和 handoff 格式，不维护两套行为定义。

## 范围

本设计包含：

- 三个协作 Skill 的职责与调用协议。
- 手动模式的角色参数、任务解析和持续会话绑定。
- Leader 模式的初始化、恢复、派发与汇合规则。
- 项目级 `.harness/` 运行目录、文件所有权和跨会话通信。
- `docs/contracts/` 正式契约与运行时快照的版本关系。
- 分层 `AGENTS.md` 的组织和协调者抽象。
- 任务及角色状态机、阻塞、决策、handoff 和模式切换。
- 迁移、验证、风险和成功标准。

本设计不包含：

- 同一时刻混合手动角色和 Leader 子代理。
- 文件系统级安全沙箱。
- 自动安装依赖或修改目标项目 `.gitignore`。
- 自动创建 Git branch、worktree、commit 或远端变更。
- 将新增 Skill 同步到本机全局目录。

## 核心原则

1. 协调职责与执行主体分离：手动模式由用户协调，Leader 模式由主 Agent 协调。
2. 共享协议只有一个来源，两个模式入口不能复制核心状态和权限规则。
3. 正式契约先冻结，角色再实施；角色不得从其他角色实现反推契约。
4. 共享事实由协调者维护，各角色只写自己的运行目录。
5. 同一任务角色在同一时刻只能有一个执行负责人。
6. 新会话必须能脱离聊天历史，通过文件恢复上下文。
7. 协调模式与 Workflow 流程预算模式正交，不能绕过流程预算、安全规则或强制 HITL。

## Skill 架构

新增三个 Skill：

```text
global/skills/
├── coordination-core/
│   ├── SKILL.md
│   ├── references/
│   │   ├── task-layout.md
│   │   ├── manifest-schema.md
│   │   ├── role-protocol.md
│   │   ├── handoff-schema.md
│   │   └── status-machine.md
│   └── agents/openai.yaml
├── coordination-manual/
│   ├── SKILL.md
│   └── agents/openai.yaml
└── coordination-leader/
    ├── SKILL.md
    └── agents/openai.yaml
```

### coordination-core

`coordination-core` 是唯一协议来源，负责：

- 任务目录布局。
- Manifest、状态、blocker、decision 和 handoff Schema。
- 正式契约发布与运行时快照规则。
- 角色路径所有权和状态转换。
- 模式切换门禁和恢复协议。

它不切换协调模式、不直接面向日常调用，也不主动派发 Agent。

### coordination-manual

`coordination-manual` 必须使用 `coordination-core`，负责：

- 解析 `role`、`action`、`root` 和 `task` 参数。
- 将当前会话持续绑定为人工协调下的指定角色。
- 初始化或加入共享任务。
- 强制手动模式不派发子代理、不虚构 Leader Agent。
- 把需要决策的事项交回用户。

### coordination-leader

`coordination-leader` 必须使用 `coordination-core`，负责：

- 初始化或恢复共享任务。
- 由当前主 Agent 承担 Leader 职责。
- 冻结契约、拆分任务、检查并行门禁和派发子代理。
- 汇总 blocker 与 handoff，检查实际变更并执行最终验证。

### 发现策略

- `coordination-manual` 和 `coordination-leader` 只能显式调用，`allow_implicit_invocation: false`。
- `coordination-core` 是依赖 Skill，界面说明通常由两个入口 Skill 使用。
- 模式入口必须以明确的必需标记引用 `coordination-core`。
- 完成兼容验证后删除旧的 `coordinating-cross-stack-sessions`，不保留旧名称别名。

## Workflow 流程预算集成

协调 Skill 不定义或复制流程档位、Agent 数量、测试轮次或澄清轮次。当前有效的 Workflow Skill 和 `global/rules/workflow-policy.md` 是流程预算的唯一事实源；协调协议只消费其判定结果。

两类状态分别回答不同问题：

- Workflow 模式决定当前任务可使用多少流程、Agent、验证和澄清预算。
- 协调模式决定由用户管理多个独立会话，还是由当前主 Agent 派发子代理。

### Leader 模式预算

- Leader 任务开始前读取当前 Workflow 模式和共享流程策略。
- 自动模式先为当前协调任务选择轻量级、中量级或重量级有效预算；固定模式直接使用对应预算。
- Leader 派发的所有只读和可读写 Agent 均按同一个任务累计计数，完成、失败、中断和返工不重置预算。
- Leader 生成或更新计划、失败分组以及每次派发前，按当前共享流程策略重新检查依赖和剩余预算。
- 协调 Skill 不以自身名义扩大预算；用户显式指定 Skill 时的预算覆盖完全遵循共享流程策略。
- 平台并发上限、路径隔离、共享契约冻结和不可绕过规则独立生效。

### 手动模式预算

- 用户创建的客户端、服务端和测试会话不是当前会话派发的子 Agent，不计入 `integration` 会话的 Agent 派发数量。
- 每个手动会话分别遵循自身当前的 Workflow 模式和任务预算，不从其他会话继承或共享剩余配额。
- `integration` 会话只记录各角色 handoff 中实际执行的验证和残余风险，不把多个会话的预算合并为虚构的全局余额。
- 协调目录、角色所有权和契约门禁在所有流程档位下均生效；流程预算不能用于绕过这些一致性要求。

## 会话状态

流程预算与协调模式分别记录：

```yaml
workflow:
  mode: auto | light | medium | heavy
  effective_budget: light | medium | heavy
  budget_scope: task

coordination:
  mode: manual-sessions | leader-subagents
  project_root: /absolute/path
  task_id: rename-character
  task_dir: /absolute/path/.harness/tasks/rename-character
  contract_version: v001

manual_binding:
  role: integration | client | server | test
  action: init | join
```

规则：

- 切换协调模式不修改 Workflow 模式或当前任务已消耗的预算。
- 新对话不继承协调模式、角色或任务绑定。
- 切换模式时清理不兼容的会话状态，但不删除任务目录或已有 handoff。
- 手动模式中用户是唯一协调负责人；`integration` 会话只提供辅助，不取得 Leader 权限。
- Leader 模式中当前主 Agent 承担协调职责。

## 命令协议

### 手动模式

```text
$coordination-manual role=<role> [action=<action>] \
root=<absolute-path> [task=<task-id>]
```

参数：

| 参数 | 必填规则 | 允许值或语义 |
|---|---|---|
| `role` | 始终必填 | `integration`、`client`、`server`、`test` |
| `action` | 可选，默认 `join` | `init`、`join` |
| `root` | 始终必填 | 已存在的项目绝对路径 |
| `task` | `init` 必填，`join` 可选 | 合法 `task-id` |

合法组合：

```text
integration + init
integration + join
client + join
server + join
test + join
```

`client/server/test + init` 非法，普通角色不能创建任务或正式契约。

### Leader 模式

```text
$coordination-leader action=<init|resume> \
root=<absolute-path> [task=<task-id>]
```

规则：

- `action` 可选；提供 `task` 时默认 `init`，未提供时默认 `resume`。
- `init` 必须显式提供 `task`。
- `resume` 可显式提供任务，也可读取当前任务指针。
- Leader 不能接管仍归属于活跃手动会话的角色。

### 任务解析

解析顺序：

1. 显式 `task=<task-id>`。
2. `<root>/.harness/current-task`。
3. 指针缺失、非法或悬空时停止并报告可用任务。

禁止通过分支名、修改时间、当前目录、聊天内容或旧会话状态猜测任务。

`task-id` 必须匹配：

```text
^[a-z0-9][a-z0-9-]{0,63}$
```

拒绝点路径、斜杠、反斜杠、空白和路径编码。规范化 `root` 后，任务目录必须位于 `<root>/.harness/tasks/` 下。

### 会话绑定

成功调用后，模式、角色、项目和任务绑定在当前会话持续生效。后续消息不必重复参数。

- 再次显式调用入口 Skill 时重新绑定。
- 更换角色、项目或任务前报告旧绑定和新绑定。
- 手动入口缺少 `role` 时不得沿用旧角色。
- 契约版本在每次写入前重新读取。
- 版本漂移时暂停并进入 `contract-changed`。

## 正式契约与运行时目录

### 两层持久化

```text
docs/contracts/<task-id>.md              # Git 跟踪，正式契约唯一可编辑来源

.harness/tasks/<task-id>/                # 本机跨会话运行时
├── manifest.yaml
├── contracts/
│   ├── current
│   ├── v001.snapshot.md
│   └── v002.snapshot.md
├── roles/
└── integration/
```

规则：

- `docs/contracts/` 是正式契约唯一可编辑来源。
- `.harness/**/contracts/` 只保存不可编辑的发布快照。
- 快照记录正式契约版本和摘要哈希，不构成第二个可编辑事实源。
- 发布新版本时先更新正式契约，经用户确认后生成新快照。
- `.harness/` 是本机运行数据，不应进入 Git。
- 目标项目未忽略 `.harness/` 时停止并通过独立 HITL 请求修改 `.gitignore`；未经授权不修改项目设置。

### 共享任务布局

```text
<project-root>/.harness/
├── current-task
└── tasks/
    └── <task-id>/
        ├── manifest.yaml
        ├── contracts/
        │   ├── current
        │   └── v001.snapshot.md
        ├── roles/
        │   ├── client/
        │   │   ├── assignment.md
        │   │   ├── status.yaml
        │   │   ├── handoff.yaml
        │   │   └── blockers/
        │   ├── server/
        │   │   ├── assignment.md
        │   │   ├── status.yaml
        │   │   ├── handoff.yaml
        │   │   └── blockers/
        │   └── test/
        │       ├── assignment.md
        │       ├── status.yaml
        │       ├── handoff.yaml
        │       └── blockers/
        └── integration/
            ├── decisions/
            ├── status.yaml
            └── report.md
```

## 文件所有权

| 路径 | 手动模式写入者 | Leader 模式写入者 |
|---|---|---|
| `.harness/current-task` | 用户授权的 `integration` 会话 | Leader |
| `manifest.yaml` | 用户授权的 `integration` 会话 | Leader |
| `contracts/**` | 用户确认后由 `integration` 发布 | Leader 经 HITL 后发布 |
| `roles/<role>/assignment.md` | `integration` 会话 | Leader |
| `roles/<role>/status.yaml` | 对应角色会话 | 对应角色子代理或 Leader 汇合 |
| `roles/<role>/handoff.yaml` | 对应角色会话 | 对应角色子代理 |
| `roles/<role>/blockers/**` | 对应角色会话 | 对应角色子代理 |
| `integration/**` | `integration` 会话 | Leader |

任何角色不得写入其他角色目录。共享文件只有协调者可以修改，避免并发会话共同写同一个状态文件。

## 分层 AGENTS.md

保留分层结构，不将角色专属规则合并到根文件：

```text
global/src/
├── AGENTS.md
├── client/
│   └── AGENTS.md
├── server/
│   └── AGENTS.md
└── test/
    └── AGENTS.md
```

职责：

- 根 `AGENTS.md`：共享契约、共同构建顺序、跨角色边界、命名规则和协调者抽象。
- `client/AGENTS.md`：Unity、UI、资源、场景和客户端可写范围。
- `server/AGENTS.md`：数据库、事务、服务注册和服务端可写范围。
- `test/AGENTS.md`：测试目录所有权、契约测试、集成测试、测试数据和清理规则。

根规则使用中性的 `coordinator`：

```text
手动模式：coordinator = user
Leader 模式：coordinator = primary-agent
```

模式、参数和状态机保留在 Skill 中，不写入角色 `AGENTS.md`。角色提示词必须显式列出根规则与对应角色规则，不能只依赖目录自动发现。

## 契约版本

- 正式契约发布后生成不可变快照。
- `contracts/current` 只保存当前版本号，例如 `v001`。
- `manifest.yaml` 记录版本、正式契约路径和摘要哈希。
- 角色开始写入前和提交 handoff 前分别读取当前版本。
- 版本变化时角色进入 `contract-changed`，停止相关写入并重新评估。
- 旧快照永久保留，用于解释历史 handoff。

## 跨会话通信

采用中心辐射模型：

```text
client/server/test
        ↓ blocker 或 handoff
用户 + integration 会话
        ↓ decision、assignment 或新契约
client/server/test
```

角色不能直接改写彼此状态，也不能通过另一角色的实现补全契约。

角色 blocker 路径：

```text
roles/<role>/blockers/<role>-<sequence>.yaml
```

协调决策路径：

```text
integration/decisions/decision-<sequence>.yaml
```

blocker 不删除，只更新 `open`、`resolved` 或 `rejected` 状态，并通过 `resolution_ref` 引用决策。涉及契约、数据、安全、路径所有权、外部副作用或项目设置的决策必须引用对应 HITL Checkpoint。

## 状态机

### 任务状态

```text
draft
  → contract-review
  → ready
  → active
  → integrating
  → completed
```

异常转换：

```text
active → blocked → active
任意未完成状态 → cancelled
```

只有协调者能修改任务状态。契约未达到 `ready` 时，角色不能进入实施状态。

### 角色状态

```text
unassigned
  → active
  → handoff-ready
  → completed
```

恢复转换：

```text
active → blocked → active
active → contract-changed → active
active → released → active
```

规则：

- `blocked`：写入 blocker 后停止相关工作。
- `contract-changed`：重新读取并评估新契约后才能继续。
- `released`：原执行者主动释放，或用户明确授权释放。
- `handoff-ready`：角色实施和验证完成，等待协调者验收。
- `completed`：协调者核对 handoff、实际文件和验证证据后确认。

## 角色所有权

角色状态记录：

```yaml
task_id: rename-character
role: client
executor_mode: manual-session
binding_id: client-session-01
state: active
contract_version: v001
updated_at: 2026-07-29T12:00:00+08:00
```

加入规则：

- `unassigned` 或已完成状态可以绑定。
- 活跃角色不能被第二个会话绑定。
- Leader 不能接管 `manual-session` 活跃角色。
- 手动会话不能接管 `leader-subagent` 活跃角色。
- 原绑定释放后，新会话才能恢复任务。
- 恢复时重新读取 Manifest、契约、角色状态、未决 blocker 和已有 handoff，不依赖聊天历史。

## Handoff

统一格式：

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
generated_at: 2026-07-29T12:00:00+08:00
```

门禁：

- `changed_files` 必须位于角色授权路径。
- `contract_version` 必须等于 `contracts/current`。
- 验证包含实际命令、退出码和结果。
- 未运行项写入 `remaining_work`。
- 存在未解决 blocker 时不能进入 `handoff-ready`。
- 空数组字段必须保留。
- 协调者检查实际文件和证据后才能标记 `completed`。

## 手动执行流程

1. 用户调用 `coordination-manual role=integration action=init`。
2. `integration` 创建任务骨架，状态为 `draft`。
3. `integration` 协助完善正式契约；用户确认后发布快照，任务进入 `ready`。
4. 用户分别创建客户端、服务端和测试会话。
5. 各会话通过 `coordination-manual role=<role>` 加入并绑定角色。
6. 角色只写自己的状态、blocker 和 handoff。
7. 用户通过 `integration` 会话处理决策、汇总结果和执行集成检查。
8. 最终完成状态必须由用户确认。

手动模式不派发子代理，不创建或假设 Leader Agent。

## Leader 执行流程

1. 用户调用 `coordination-leader action=init|resume`。
2. Leader 读取当前 Workflow 模式和共享流程策略，确定当前任务的有效预算与已消耗额度。
3. Leader 初始化或读取共享任务。
4. Leader 冻结正式契约并发布运行时快照。
5. Leader 根据依赖、路径所有权、并行规则和剩余 Agent 预算派发子代理。
6. 子代理使用与手动角色相同的状态、blocker 和 handoff 协议。
7. Leader 汇合结果，核对实际改动和共享契约。
8. Leader 在当前验证预算内执行最小充分的整体验证并报告证据；不可绕过的必要验证不因预算不足而静默省略。

每个子任务 prompt 必须包含绝对工作区、适用规则、契约版本、写入边界、验收条件和禁止继续派发条款。

## 模式切换

### 手动切换为 Leader

仅当所有手动角色处于以下状态之一时允许：

```text
unassigned | released | handoff-ready | completed
```

Leader 读取现有状态，不重新初始化或覆盖任务文件。存在活跃手动绑定时必须进入 HITL，不能抢占。

### Leader 切换为手动

1. Leader 停止派发新任务。
2. 等待已派发子代理完成，或在安全边界停止。
3. 将角色状态写为 `released` 或 `handoff-ready`。
4. 写入集成汇总和剩余工作。
5. 用户再创建手动角色会话。

### 第一版限制

不支持同一时刻按角色混合两种执行方式。允许整个任务按阶段切换协调模式，但不能出现客户端由手动会话执行、服务端同时由 Leader 子代理执行的混合所有权。

## 错误处理

- 参数缺失、重复、未知或非法时停止并展示正确调用格式。
- 同名任务已存在时 `init` 停止，不覆盖。
- 当前任务指针悬空时停止，不猜测目标任务。
- 契约未冻结时角色停止，不生成自定义字段。
- 路径授权重叠时停止并交由协调者重新划分。
- 角色所有权冲突时停止，不抢占。
- 契约哈希不一致时进入 `contract-changed`。
- `.harness` 未被忽略时进入 HITL，不直接修改 `.gitignore`。
- 会话中断时保留所有运行文件，通过显式释放和重新绑定恢复。

## AGENTS.md 迁移

修改现有 `global/src/AGENTS.md`：

- 将固定 `leader` 抽象为 `coordinator`。
- 说明手动模式由用户协调，Leader 模式由主 Agent 协调。
- 保留共享构建、协议、命名和跨角色规则。
- 正式契约继续位于 `docs/contracts/`。
- 修正现有角色表格缺失的结束符。
- 删除或补全空的“项目约束”章节，不保留空标题。

保留 `global/src/client/AGENTS.md` 和 `global/src/server/AGENTS.md` 的专属约束，只更新其中对固定 `leader` 的引用。

新增 `global/src/test/AGENTS.md`，定义：

- `Src/Tests/` 写入所有权。
- Client、Server、Lib 和正式契约默认只读。
- 契约测试、集成测试和端到端测试边界。
- 测试数据初始化、隔离和清理要求。
- 不得修改生产实现来迁就测试。

## 迁移顺序

1. 创建并验证 `coordination-core`。
2. 创建并验证 `coordination-manual`。
3. 创建并验证 `coordination-leader`。
4. 调整分层 `AGENTS.md` 并新增测试规则。
5. 运行手动与 Leader 兼容场景。
6. 删除旧 `coordinating-cross-stack-sessions`。
7. 检查不存在旧 Skill 引用。

不修改或同步本机全局 Skill 目录。

## 验证策略

### 参数与路径

- 验证完整参数、缺失参数、未知参数和重复参数。
- 验证非法角色和非法 `role/action` 组合。
- 验证合法、非法、越界和编码后的 `task-id`。
- 验证显式任务、有效指针、缺失指针和悬空指针。

### 协议与所有权

- 验证普通角色不能初始化任务或修改正式契约。
- 验证同一角色不能被两个会话或两种模式同时占用。
- 验证角色只能写自身目录。
- 验证 blocker、decision 和 handoff Schema。
- 验证契约版本与哈希漂移进入 `contract-changed`。

### 模式行为

- 验证手动模式不派发 Agent、不虚构 Leader。
- 验证 Leader 模式生成自包含子任务并复核实际结果。
- 验证模式切换保留任务状态且不抢占活跃角色。
- 验证第一版拒绝同时混合执行。
- 验证新会话不依赖聊天历史恢复。

### 流程预算兼容性

- 验证协调模式不修改 `auto`、`light`、`medium` 或 `heavy` Workflow 状态。
- 验证自动模式先确定当前任务有效预算，再进入 Leader 派发流程。
- 验证 Leader 派发按任务累计计入只读和可读写 Agent 预算，失败、中断和返工不重置计数。
- 验证手动创建的独立会话不计入 `integration` 会话的派发数，各会话分别遵循自身预算。
- 验证协调 Skill 不复制 `workflow-policy.md` 中的数值配额或重新引入用户可见复杂度分级。
- 验证流程预算不足时遵循共享策略的覆盖、升级或残余风险规则，不由协调 Skill 自创例外。

### Skill 质量

- 使用官方 Skill 校验器检查三个 Skill。
- 运行无 Skill 基线和带 Skill 应用测试。
- 用全新只读 Agent 测试角色规则、所有权、契约漂移和模式切换。
- 检查两个入口 Skill 不复制核心协议。
- 检查旧 Skill 删除后无残留引用。

若官方校验器因缺少 `PyYAML` 无法运行，不安装依赖；记录阻塞并使用独立 YAML 解析与结构检查作为降级证据。

## 成功标准

人工协调者可以初始化：

```text
$coordination-manual role=integration action=init \
task=rename-character root=/Projects/MMORPG
```

随后三个独立会话可以加入当前任务：

```text
$coordination-manual role=client root=/Projects/MMORPG
$coordination-manual role=server root=/Projects/MMORPG
$coordination-manual role=test root=/Projects/MMORPG
```

同一任务也可以在所有手动角色安全释放后切换为：

```text
$coordination-leader action=resume root=/Projects/MMORPG
```

两种方式读取同一正式契约、运行时快照和状态文件，不同时占用同一角色，并能在无聊天历史的新会话中恢复。

## 风险

- Skill 是行为约束，不是文件系统安全边界。
- 本机共享目录可能被外部进程修改，因此必须校验版本、哈希和角色所有权。
- 独立 worktree 不自动共享各自根目录中的 `.harness`；`root` 必须指向所有会话共同可访问的协调项目根，角色代码工作区另由 Manifest 记录。
- 无法自动判断中断会话是否已经永久退出，角色接管必须由用户显式释放。
- 正式契约和运行快照需要发布流程保证哈希一致。
- 第一版不支持同一时刻混合手动角色与 Leader 子代理。
