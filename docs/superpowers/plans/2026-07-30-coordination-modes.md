# 跨角色协调模式实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 实现共享协调协议、人工多会话入口和 Leader 子代理入口，使两种开发方式通过同一正式契约、运行目录、角色所有权和 handoff 协作。

**架构：** `coordination-core` 是任务布局、Schema、状态机和权限协议的唯一事实源；`coordination-manual` 与 `coordination-leader` 只实现各自的模式切换和执行流程。正式契约保存在 `docs/contracts/`，本机跨会话状态保存在项目根目录 `.harness/`，分层 `global/src/**/AGENTS.md` 提供共享及角色专属约束。

**技术栈：** Markdown、YAML、Codex Skill、Shell/Ruby 静态校验、Superpowers Skill 场景测试、Git

---

## 全局约束

- 规格来源：`docs/superpowers/specs/2026-07-29-coordination-modes-design.md`。
- 流程预算来源：`global/rules/workflow-policy.md`；协调 Skill 不复制其中的数值配额。
- 使用 `skill-creator` 和 `superpowers-zh:writing-skills` 创建、测试和验证每个 Skill。
- 每个 Skill 按“无 Skill 基线 → 最小 Skill → 应用测试 → 修订 → 复测”顺序独立完成；前一个 Skill 验证完成后才能创建下一个。
- 不创建 worktree、branch 或 tag，不推送远端。
- 不修改或同步 `/Users/edy/.agents/skills/`、`/Users/edy/.codex/skills/` 等本机全局 Skill 目录。
- 不恢复当前工作树中已删除的 `global/root/**`、`global/rules/codegraph-navigation.md` 或 `global/rules/parallel-execution.md`。
- 保留用户及其他任务对 `global/AGENTS.md`、`global/rules/**` 和 Workflow Skill 的未提交修改。
- `global/src/**` 当前属于未跟踪目录；提交前必须精确暂存本计划列出的文件，不能顺带暂存 `.DS_Store`。
- 不安装 `PyYAML`。官方 `quick_validate.py` 无法运行时，保留失败证据并执行本计划给出的 Ruby YAML 降级校验。
- 实现期间不在此 harness 仓库创建真实 `.harness/` 任务目录；场景测试只读检查生成的协议文本，临时文件使用 `mktemp -d`，完成后移入系统废纸篓或保留路径并报告。

## 文件结构

### 新建

- `global/skills/coordination-core/SKILL.md`：共享协议入口和引用路由。
- `global/skills/coordination-core/agents/openai.yaml`：共享协议 Skill 元数据。
- `global/skills/coordination-core/references/task-layout.md`：正式契约与 `.harness/` 目录、路径安全和文件所有权。
- `global/skills/coordination-core/references/manifest-schema.md`：Manifest、契约发布信息和 Workflow 预算引用 Schema。
- `global/skills/coordination-core/references/role-protocol.md`：角色绑定、读写边界、blocker 和 decision 协议。
- `global/skills/coordination-core/references/handoff-schema.md`：统一 handoff Schema 与完成门禁。
- `global/skills/coordination-core/references/status-machine.md`：任务、角色状态机和模式切换。
- `global/skills/coordination-manual/SKILL.md`：人工多会话参数解析、持续绑定、初始化和加入流程。
- `global/skills/coordination-manual/agents/openai.yaml`：人工模式 Skill 元数据。
- `global/skills/coordination-leader/SKILL.md`：Leader 初始化、恢复、预算检查、派发和汇合流程。
- `global/skills/coordination-leader/agents/openai.yaml`：Leader 模式 Skill 元数据。
- `global/src/test/AGENTS.md`：测试角色职责、写入范围、数据隔离和验证约束。

### 修改

- `global/src/AGENTS.md`：将固定 `leader` 改为中性 `coordinator`，修复角色表格和空章节。
- `global/src/client/AGENTS.md`：将跨端契约交接对象从固定 `leader` 改为当前 `coordinator`。
- `global/src/server/AGENTS.md`：将跨端契约交接对象从固定 `leader` 改为当前 `coordinator`。

### 删除

- `global/skills/coordinating-cross-stack-sessions/SKILL.md`
- `global/skills/coordinating-cross-stack-sessions/agents/openai.yaml`
- `global/skills/coordinating-cross-stack-sessions/references/client-prompt.md`
- `global/skills/coordinating-cross-stack-sessions/references/server-prompt.md`
- `global/skills/coordinating-cross-stack-sessions/references/test-prompt.md`

旧 Skill 只能在三个新 Skill、分层规则和兼容场景全部验证通过后删除。

---

### 任务 1：建立协调协议的失败基线

**文件：**
- 读取：`docs/superpowers/specs/2026-07-29-coordination-modes-design.md`
- 读取：`global/rules/workflow-policy.md`
- 读取：`global/src/AGENTS.md`
- 读取：`global/src/client/AGENTS.md`
- 读取：`global/src/server/AGENTS.md`
- 不修改文件

- [ ] **步骤 1：确认基线时三个新 Skill 均不存在**

运行：

```bash
test ! -e global/skills/coordination-core/SKILL.md
test ! -e global/skills/coordination-manual/SKILL.md
test ! -e global/skills/coordination-leader/SKILL.md
```

预期：三个命令退出码均为 `0`。

- [ ] **步骤 2：运行手动模式无 Skill 基线场景**

派发一个全新只读 Agent，提示词必须只提供规格中的目标，不提供拟实现的 Skill 内容：

```text
[MICRO-TASK] 目标：设计一次人工多会话角色加入操作。输入为 role=client、root=/Projects/MMORPG，task 省略；项目存在 .harness/current-task。请说明你会读取哪些文件、如何绑定角色、允许写哪些路径，以及契约版本变化时如何处理。/ 范围：只读当前仓库已有规则，不读取任何 coordination-* Skill。/ 依赖摘要：用户是协调者，不存在 Leader Agent。/ 期望输出格式：操作步骤、状态写入、阻塞条件。/ 禁止事项：不修改文件、不派发、不假设聊天历史。
```

逐字记录是否出现以下失败：

- 根据聊天历史或当前目录猜测任务。
- 未检查 `.harness/current-task`。
- 未检查角色已有活跃绑定。
- 将用户或当前会话虚构为 Leader Agent。
- 未在写入前后检查契约版本。

- [ ] **步骤 3：运行 Leader 模式无 Skill 基线场景**

派发另一个全新只读 Agent：

```text
[MICRO-TASK] 目标：恢复一个跨端任务并派发 client、server、test 子任务。/ 范围：只读当前仓库已有规则，不读取任何 coordination-* Skill。/ 依赖摘要：当前 Workflow 为 auto，任务内已有一个失败的只读 Agent 和一个 active 的 manual-session client 绑定。/ 期望输出格式：预算处理、所有权检查、派发决策、汇合门禁。/ 禁止事项：不修改文件、不实际派发、不忽略现有绑定。
```

逐字记录是否出现以下失败：

- 未把失败 Agent 计入累计预算。
- 抢占活跃手动角色。
- 契约未冻结就派发。
- 子任务未包含绝对路径、规则、契约、验收和禁止继续派发。
- 以子代理完成声明替代最终验证。

- [ ] **步骤 4：汇总基线失败清单**

在当前会话记录精确原文；后续三个 Skill 只解决已经观察到的失败和规格明确要求，不加入无需求依据的功能。

预期：至少确认人工模式和 Leader 模式各一项缺失行为；若基线意外完全合规，使用更具压力的场景复测一次，仍合规则记录“现有规则已覆盖”，但继续实现用户明确要求的可复用入口。

---

### 任务 2：创建并验证 coordination-core

**文件：**
- 创建：`global/skills/coordination-core/SKILL.md`
- 创建：`global/skills/coordination-core/agents/openai.yaml`
- 创建：`global/skills/coordination-core/references/task-layout.md`
- 创建：`global/skills/coordination-core/references/manifest-schema.md`
- 创建：`global/skills/coordination-core/references/role-protocol.md`
- 创建：`global/skills/coordination-core/references/handoff-schema.md`
- 创建：`global/skills/coordination-core/references/status-machine.md`

- [ ] **步骤 1：使用官方初始化脚本创建 Skill 骨架**

运行：

```bash
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/init_skill.py coordination-core \
  --path /Users/edy/Desktop/my_harness/global/skills \
  --resources references \
  --interface 'display_name=跨角色协调协议' \
  --interface 'short_description=定义任务目录、契约版本、角色所有权和跨会话交接协议' \
  --interface 'default_prompt=使用 $coordination-core 检查当前跨角色任务的协议和状态。'
```

预期：输出 `[OK] Skill 'coordination-core' initialized successfully`。若脚本无执行权限，仍使用上述 `python3` 调用，不修改脚本权限。

- [ ] **步骤 2：编写最小 SKILL.md**

`global/skills/coordination-core/SKILL.md` 必须采用以下结构，不复制引用文件的详细 Schema：

```markdown
---
name: coordination-core
description: Use when manual Codex sessions or a leader Agent need a shared cross-role task protocol, including persistent contracts, role ownership, blockers, handoffs, recovery, or coordination-mode transitions.
---

# 跨角色协调协议

## 核心边界

本 Skill 是协调协议的唯一事实源，不切换协调模式，不自行派发 Agent。
正式契约位于 `docs/contracts/`；`.harness/` 只保存本机运行状态和不可编辑快照。

## 必读引用

按当前操作完整读取：

- 初始化、路径或文件权限：[references/task-layout.md](references/task-layout.md)
- Manifest 或契约发布：[references/manifest-schema.md](references/manifest-schema.md)
- 角色绑定、blocker 或 decision：[references/role-protocol.md](references/role-protocol.md)
- Handoff 或完成验收：[references/handoff-schema.md](references/handoff-schema.md)
- 状态转换、恢复或模式切换：[references/status-machine.md](references/status-machine.md)

## 不可绕过门禁

- 不从聊天历史、分支、目录时间或当前工作目录猜测任务。
- 不允许两个执行者同时拥有同一角色。
- 不允许角色修改正式契约、其他角色目录或集成目录。
- 每次写入前后核对当前契约版本。
- Workflow 预算由当前模式和 `global/rules/workflow-policy.md` 定义，本 Skill 不复制配额。
- `.harness/` 未被目标项目忽略时，按 HITL 请求授权，不自行修改 `.gitignore`。
```

- [ ] **步骤 3：编写 task-layout.md**

文件必须完整定义：

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
<root>/.harness/tasks/<task-id>/integration/status.yaml
<root>/.harness/tasks/<task-id>/integration/report.md
```

同时写明：

- `task-id` 正则为 `^[a-z0-9][a-z0-9-]{0,63}$`。
- `root` 必须是已存在的规范化绝对路径。
- 解析后的任务目录必须位于 `<root>/.harness/tasks/`。
- `docs/contracts/` 是唯一可编辑正式契约。
- 快照不可原地修改，版本递增为 `v001`、`v002`。
- 角色只写自己的 `status.yaml`、`handoff.yaml` 和 `blockers/`。
- 协调者写共享文件、assignment 和 `integration/`。

- [ ] **步骤 4：编写 manifest-schema.md**

定义以下完整示例，并逐字段说明必填、枚举和更新者：

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

明确 `effective_budget` 是当前任务的消费结果，不在协调模式切换时重置；数值配额不得写进 Manifest Schema。

- [ ] **步骤 5：编写 role-protocol.md**

定义：

- `integration`、`client`、`server`、`test` 的读写矩阵。
- `binding_id` 创建、占用、释放和恢复规则。
- `executor_mode` 只能为 `manual-session` 或 `leader-subagent`。
- blocker 文件名为 `<role>-<sequence>.yaml`。
- decision 文件名为 `decision-<sequence>.yaml`。

Blocker 示例必须完整：

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

Decision 示例必须完整：

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

- [ ] **步骤 6：编写 handoff-schema.md**

采用规格中的完整 handoff 示例，并增加以下硬性检查：

- `status` 必须为 `handoff-ready`。
- `changed_files` 全部位于角色授权路径。
- `contract_version` 等于 `contracts/current`。
- `verification` 每项含 `command`、`exit_code`、`result`。
- 未执行验证进入 `remaining_work`。
- `blockers_resolved`、`risks`、`remaining_work` 即使为空也保留数组。
- 协调者复核实际变更前不得把角色标记为 `completed`。

- [ ] **步骤 7：编写 status-machine.md**

任务状态必须完整定义：

```text
draft → contract-review → ready → active → integrating → completed
active → blocked → active
任意未完成状态 → cancelled
```

角色状态必须完整定义：

```text
unassigned → active → handoff-ready → completed
active → blocked → active
active → contract-changed → active
active → released → active
```

定义非法转换、更新者、恢复前置条件，以及手动与 Leader 切换时允许的角色状态集合：

```text
unassigned | released | handoff-ready | completed
```

- [ ] **步骤 8：校验 coordination-core 结构**

先运行官方校验器：

```bash
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py \
  global/skills/coordination-core
```

预期：输出 `Skill is valid!`。若因 `ModuleNotFoundError: No module named 'yaml'` 失败，记录该输出，然后运行：

```bash
ruby -ryaml -e '
s = File.read(ARGV.fetch(0), encoding: "UTF-8")
m = YAML.safe_load(s.split("---\n", 3).fetch(1))
abort "invalid" unless m.keys.sort == %w[description name]
abort "invalid name" unless m["name"] == "coordination-core"
abort "invalid description" unless m["description"].start_with?("Use when")
puts "coordination-core frontmatter OK"
' global/skills/coordination-core/SKILL.md
```

再运行：

```bash
rg -n 'TODO|TBD|待定|Trivial|Standard|Complex|workflow-full' \
  global/skills/coordination-core
```

预期：无输出，`rg` 退出码为 `1`。

- [ ] **步骤 9：运行 coordination-core 应用测试**

使用全新只读 Agent，提示：

```text
使用 $coordination-core 检查以下状态：task=rename-character，client 已被 manual-session 绑定，Leader 准备 resume，契约 current 从 v001 变为 v002。只报告允许的状态转换、禁止的所有权动作、需要读取的文件和下一步。
```

预期必须包含：

- Leader 不得抢占 client。
- client 进入 `contract-changed`。
- 重新读取正式契约、快照、Manifest 和角色状态。
- 不能重置 Workflow 已消耗预算。

- [ ] **步骤 10：提交 coordination-core**

```bash
git add global/skills/coordination-core
git diff --cached --check
git diff --cached --name-status
git commit -m "feat(协调): 添加跨角色共享协议"
```

预期：暂存区只包含 `coordination-core` 的 7 个新文件。

---

### 任务 3：创建并验证 coordination-manual

**文件：**
- 创建：`global/skills/coordination-manual/SKILL.md`
- 创建：`global/skills/coordination-manual/agents/openai.yaml`
- 依赖：`global/skills/coordination-core/**`

- [ ] **步骤 1：初始化手动入口 Skill**

```bash
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/init_skill.py coordination-manual \
  --path /Users/edy/Desktop/my_harness/global/skills \
  --interface 'display_name=人工多会话协调' \
  --interface 'short_description=将当前会话绑定为人工协调下的客户端、服务端、测试或集成角色' \
  --interface 'default_prompt=使用 $coordination-manual role=client root=/Projects/MMORPG 加入当前任务。'
```

预期：输出初始化成功，且不创建无用的 `references/`。

- [ ] **步骤 2：编写参数和持续绑定协议**

`SKILL.md` 必须包含：

```text
$coordination-manual role=<integration|client|server|test> \
  [action=<init|join>] root=<absolute-path> [task=<task-id>]
```

并明确：

- **必需子技能：** 使用 `coordination-core`。
- `role`、`root` 始终必填，不从旧会话绑定继承缺失参数。
- `action` 默认 `join`。
- `integration + init`、`integration + join`、`client/server/test + join` 合法。
- `client/server/test + init` 非法。
- `init` 必须显式提供 `task`。
- `join` 未提供 `task` 时读取 `<root>/.harness/current-task`。
- 手动模式中用户是协调者，当前 Agent 只承担指定角色，不派发子代理。
- 成功绑定后在当前会话持续生效；新对话复位。
- 切换角色、项目或任务时先报告旧绑定和新绑定，再校验所有权。

- [ ] **步骤 3：编写 integration/init 流程**

顺序必须固定：

1. 校验参数和目标路径。
2. 确认同名任务不存在。
3. 检查目标项目是否已忽略 `.harness/`。
4. 未忽略时发布 HITL Checkpoint，不修改 `.gitignore`。
5. 创建任务骨架，状态为 `draft`。
6. 创建或读取 `docs/contracts/<task-id>.md`。
7. 契约经 HITL 确认后生成不可变快照、哈希和 `contracts/current`。
8. 将任务置为 `ready` 并更新 `.harness/current-task`。

禁止在契约未确认时创建 `ready` 状态或角色 assignment。

- [ ] **步骤 4：编写角色 join 流程**

顺序必须固定：

1. 解析任务。
2. 读取 Manifest、当前快照和对应角色 assignment。
3. 完整读取 Manifest 列出的祖先链与显式角色 `AGENTS.md`。
4. 检查角色状态和 `executor_mode`。
5. 创建当前会话 `binding_id`，写入 `active`。
6. 写入前后检查 `contracts/current`。
7. 只在角色授权路径和自身运行目录工作。

遇到契约缺口、越界需求或外部副作用时写 blocker 并停止相关工作。

- [ ] **步骤 5：运行参数静态检查**

```bash
rg -q 'role=<integration|client|server|test>' global/skills/coordination-manual/SKILL.md
rg -q 'action=<init|join>' global/skills/coordination-manual/SKILL.md
rg -q 'current-task' global/skills/coordination-manual/SKILL.md
rg -q '不派发' global/skills/coordination-manual/SKILL.md
rg -q 'coordination-core' global/skills/coordination-manual/SKILL.md
```

预期：全部退出码为 `0`。

- [ ] **步骤 6：运行手动模式应用测试**

用全新只读 Agent 分别执行三个场景：

```text
使用 $coordination-manual，参数 role=client root=/Projects/MMORPG，task 省略，current-task 指向 rename-character，client 状态 unassigned。报告完整 join 顺序。
```

预期：解析指针、读取 Manifest/契约/规则、绑定 client，不派发 Agent。

```text
使用 $coordination-manual，参数 role=server action=init root=/Projects/MMORPG task=rename-character。
```

预期：拒绝非法组合，不创建任何目录。

```text
使用 $coordination-manual，参数 role=test root=/Projects/MMORPG；test 已被另一个 manual-session active 绑定。
```

预期：拒绝抢占并返回用户协调，不自行释放旧绑定。

- [ ] **步骤 7：校验并提交 coordination-manual**

执行官方或任务 2 的 Ruby 降级校验，将期望名称改为 `coordination-manual`。然后：

```bash
git add global/skills/coordination-manual
git diff --cached --check
git diff --cached --name-status
git commit -m "feat(协调): 添加人工多会话模式"
```

预期：只提交手动入口的 `SKILL.md` 与 `agents/openai.yaml`。

---

### 任务 4：创建并验证 coordination-leader

**文件：**
- 创建：`global/skills/coordination-leader/SKILL.md`
- 创建：`global/skills/coordination-leader/agents/openai.yaml`
- 依赖：`global/skills/coordination-core/**`
- 依赖：`global/rules/workflow-policy.md`
- 读取：`global/rules/multi-agent-assistance.md`

- [ ] **步骤 1：初始化 Leader 入口 Skill**

```bash
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/init_skill.py coordination-leader \
  --path /Users/edy/Desktop/my_harness/global/skills \
  --interface 'display_name=Leader 子代理协调' \
  --interface 'short_description=由当前主 Agent 统筹契约、派发角色子代理并汇总验证' \
  --interface 'default_prompt=使用 $coordination-leader action=resume root=/Projects/MMORPG 恢复当前任务。'
```

- [ ] **步骤 2：编写参数与模式切换协议**

必须支持：

```text
$coordination-leader [action=<init|resume>] \
  root=<absolute-path> [task=<task-id>]
```

规则：

- **必需子技能：** 使用 `coordination-core`。
- 提供 `task` 时默认 `init`，未提供时默认 `resume`。
- `init` 必须显式提供 `task`。
- `resume` 未提供任务时读取 `current-task`。
- 当前主 Agent 是 Leader；子 Agent 不得继续派发。
- Leader 切换不能修改 Workflow 模式或重置已消耗预算。
- 存在 `manual-session + active` 角色时停止并发布 HITL，不抢占。

- [ ] **步骤 3：编写 Workflow 预算消费规则**

明确要求：

1. 读取当前 Workflow Skill 和 `global/rules/workflow-policy.md`。
2. `auto` 在当前任务开始时先确定有效预算。
3. 已完成、失败、中断和返工 Agent 均计入任务累计派发。
4. 只读 Agent 不得升级为可读写 Agent。
5. 生成或更新计划、失败分组和每次派发前重建依赖层并检查剩余预算。
6. 协调 Skill 不复制数值配额，也不自行创建预算例外。
7. 用户显式 Skill 覆盖、自动升级和固定模式残余风险完全服从 `workflow-policy.md`。

- [ ] **步骤 4：编写派发模板**

每个角色子任务 prompt 必须包含以下字段：

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

提示词不得使用“上文”“同前”或隐含聊天历史。

- [ ] **步骤 5：编写汇合和完成门禁**

Leader 必须：

- 比较实际修改路径、角色授权路径和 handoff 声明。
- 检查三方使用相同契约版本和哈希。
- 发现子代理擅改共享契约时拒绝集成。
- 按依赖顺序处理 handoff，不让角色自行集成。
- 在剩余验证预算内运行最小充分验证。
- 不可绕过验证与预算冲突时按 Workflow 策略处理，不静默省略。
- 只有实际证据支持时才能把任务置为 `completed`。

- [ ] **步骤 6：运行 Leader 模式应用测试**

用全新只读 Agent 运行：

```text
使用 $coordination-leader action=resume root=/Projects/MMORPG。当前 Workflow=medium，本任务已使用 1 个失败只读 Agent；client 为 active manual-session，server/test unassigned；契约 v001 已冻结。报告派发决策。
```

预期：

- 失败 Agent 计入预算。
- 因 client 活跃手动绑定停止整任务模式接管，不派发 server/test 形成混合模式。
- 输出需要用户释放或完成 client 的 HITL。

再运行：

```text
使用 $coordination-leader action=resume root=/Projects/MMORPG。所有角色 released，契约 v001 已冻结，client/server 写入范围互斥，test 依赖两者交付。输出依赖层和派发提示词必填字段。
```

预期：先并行层 `client + server`，后串行 `test`，最后 Leader 集成；输出完整自包含字段。

- [ ] **步骤 7：校验并提交 coordination-leader**

执行官方或 Ruby 降级校验，并确认：

```bash
rg -q 'workflow-policy.md' global/skills/coordination-leader/SKILL.md
rg -q '累计' global/skills/coordination-leader/SKILL.md
rg -q 'manual-session' global/skills/coordination-leader/SKILL.md
rg -q '不继续派发' global/skills/coordination-leader/SKILL.md
```

然后：

```bash
git add global/skills/coordination-leader
git diff --cached --check
git diff --cached --name-status
git commit -m "feat(协调): 添加 Leader 子代理模式"
```

---

### 任务 5：迁移分层 AGENTS.md

**文件：**
- 修改：`global/src/AGENTS.md`
- 修改：`global/src/client/AGENTS.md`
- 修改：`global/src/server/AGENTS.md`
- 创建：`global/src/test/AGENTS.md`

- [ ] **步骤 1：运行迁移前契约检查并确认失败**

```bash
rg -q '\| `leader` \|' global/src/AGENTS.md
rg -q '交由 `leader`' global/src/client/AGENTS.md
rg -q '经 `leader`' global/src/server/AGENTS.md
test ! -e global/src/test/AGENTS.md
```

预期：全部退出码为 `0`，证明旧固定 Leader 语义和测试规则缺失仍存在。

- [ ] **步骤 2：修改根协调者规则**

把角色表中的固定 Leader 行替换为：

```markdown
| `coordinator` | 唯一协调者：维护 `docs/contracts/`、划分角色所有权、处理 blocker、汇总 handoff 和最终验证；手动模式由用户承担，Leader 模式由当前主 Agent 承担。 |
```

同时：

- 补齐表格行结尾 `|`。
- 保留 `client`、`server`、`lib`、`test` 行。
- 将“交由 leader”的共享措辞改为“交由当前 coordinator”。
- 删除空的 `## 项目约束` 标题，不新增无内容章节。
- 保留共享构建、C# 版本、命名和 `message.proto` 规则。

- [ ] **步骤 3：修改客户端和服务端引用**

客户端改为：

```markdown
- 只读：`Src/Lib/`、`Src/Server/`、`Src/Tests/`；跨端接口先交由当前 `coordinator` 固化到 `docs/contracts/`。
```

服务端改为：

```markdown
- 可写：`Src/Server/`；只读：`Src/Client/`、`Src/Lib/`、`Src/Tests/`。跨端接口先经当前 `coordinator` 稳定到 `docs/contracts/`。
```

不得修改其余 Unity、数据库、生成代码或注册规则。

- [ ] **步骤 4：创建测试角色规则**

`global/src/test/AGENTS.md` 写入：

```markdown
# Test Agent（测试）

## 职责与范围

- 负责 `Src/Tests/` 中的契约测试、客户端和服务端集成测试及端到端验证。
- 默认仅可写 `Src/Tests/`；`Src/Client/`、`Src/Server/`、`Src/Lib/` 和 `docs/contracts/` 只读。
- 以已冻结的正式契约和验收条件为预期来源，不根据当前实现降低或改写断言。
- 发现生产缺陷时写入证据和归属建议，不修改生产实现来迁就测试。

## 测试数据

- 每项测试明确前置状态、数据隔离方式和清理步骤。
- 测试必须可重复运行；不能自动清理的数据在 handoff 中列为风险和剩余工作。
- 不连接或修改未获授权的共享数据库、外部服务或生产数据。

## 验证与交接

- 按风险覆盖契约、成功、错误、鉴权、边界、重复、并发、超时、回滚和持久化一致性；不适用项说明依据。
- Handoff 记录实际命令、退出码、关键结果、未运行项及原因。
- 不自行修改正式契约、集成其他角色改动或继续派发子任务。
```

- [ ] **步骤 5：运行角色规则契约检查**

```bash
! rg -n '\| `leader` \||交由 `leader`|经 `leader`' global/src -g 'AGENTS.md'
rg -q '\| `coordinator` \|' global/src/AGENTS.md
rg -q '手动模式由用户承担' global/src/AGENTS.md
rg -q 'Leader 模式由当前主 Agent 承担' global/src/AGENTS.md
rg -q '默认仅可写 `Src/Tests/`' global/src/test/AGENTS.md
rg -q '不修改生产实现来迁就测试' global/src/test/AGENTS.md
```

预期：全部退出码为 `0`。

- [ ] **步骤 6：提交分层规则迁移**

```bash
git add global/src/AGENTS.md global/src/client/AGENTS.md \
  global/src/server/AGENTS.md global/src/test/AGENTS.md
git diff --cached --check
git diff --cached --name-status
git commit -m "refactor(协调): 抽象跨角色协调者"
```

预期：不暂存 `global/src/.DS_Store`。

---

### 任务 6：运行跨模式兼容验证并删除旧 Skill

**文件：**
- 删除：`global/skills/coordinating-cross-stack-sessions/**`
- 验证：`global/skills/coordination-core/**`
- 验证：`global/skills/coordination-manual/**`
- 验证：`global/skills/coordination-leader/**`
- 验证：`global/src/**/AGENTS.md`

- [ ] **步骤 1：运行手动初始化到角色交付的完整场景**

使用全新只读 Agent，以以下输入使用新 Skill：

```text
$coordination-manual role=integration action=init task=rename-character root=/Projects/MMORPG
```

提供已确认 `.harness/` 被忽略、正式契约 v001 已冻结的事实。要求输出初始化文件清单和状态，不实际写入 `/Projects`。

预期：任务从 `draft` 经 `contract-review` 到 `ready`，生成不可变快照、哈希和当前指针。

再分别模拟 `client`、`server`、`test` join，预期各自只写自身运行目录，并在 handoff 前复核契约版本。

- [ ] **步骤 2：运行模式切换场景**

场景一：`client=active manual-session` 时调用 Leader resume。

预期：停止并请求用户处理，不派发任何角色。

场景二：所有角色为 `released`，调用 Leader resume。

预期：保留现有 Manifest、快照、blocker 和 handoff；不重新初始化；按 Workflow 预算和依赖层派发。

场景三：Leader 子代理仍 active 时调用手动 client join。

预期：拒绝手动绑定，不形成混合执行。

- [ ] **步骤 3：运行契约漂移和恢复场景**

输入：角色绑定 `v001`，`contracts/current=v002`。

预期：角色进入 `contract-changed`，停止相关写入，读取正式契约、v002 快照、Manifest、未决 blocker 和旧 handoff 后重新评估。

- [ ] **步骤 4：运行全量静态校验**

```bash
for skill in coordination-core coordination-manual coordination-leader; do
  ruby -ryaml -e '
  s = File.read(ARGV.fetch(0), encoding: "UTF-8")
  m = YAML.safe_load(s.split("---\n", 3).fetch(1))
  abort "invalid" unless m.keys.sort == %w[description name]
  abort "invalid description" unless m["description"].start_with?("Use when")
  puts "#{m["name"]} OK"
  ' "global/skills/$skill/SKILL.md"
  ruby -ryaml -e '
  m = YAML.safe_load(File.read(ARGV.fetch(0), encoding: "UTF-8"))
  i = m.fetch("interface")
  abort "invalid" unless %w[display_name short_description default_prompt].all? { |k| i[k].is_a?(String) && !i[k].empty? }
  puts "openai.yaml OK"
  ' "global/skills/$skill/agents/openai.yaml"
done
```

预期：三个 Skill 和三个 `openai.yaml` 均输出 `OK`。

运行：

```bash
if rg -n 'TODO|TBD|待定|Trivial|Standard|Complex|workflow-full' \
  global/skills/coordination-core \
  global/skills/coordination-manual \
  global/skills/coordination-leader; then
  exit 1
fi

if rg -n '每个任务累计最多|测试轮次.*最多|澄清轮次.*最多' \
  global/skills/coordination-core \
  global/skills/coordination-manual \
  global/skills/coordination-leader; then
  exit 1
fi
```

预期：无输出，退出码为 `0`，证明协调 Skill 未复制旧分级或数值预算。

- [ ] **步骤 5：删除旧 Skill**

使用 `apply_patch` 删除旧目录中的五个文件，不使用递归删除命令。删除后运行：

```bash
test ! -e global/skills/coordinating-cross-stack-sessions/SKILL.md
! rg -n 'coordinating-cross-stack-sessions' global docs \
  -g '*.md' -g '*.yaml'
```

预期：旧 Skill 不存在且有效文件无残留引用。历史 Git 提交不在搜索范围内。

- [ ] **步骤 6：检查完整修改边界**

```bash
git status --short
git diff --check
git diff --stat
```

逐项确认：

- 新增三个 Skill。
- 修改三份现有角色规则并新增测试规则。
- 删除旧协调 Skill。
- 未修改 Workflow Skill 或 `workflow-policy.md`。
- 未恢复已删除的旧规则和 `global/root/**`。
- 未暂存或提交 `.DS_Store`。
- 未同步本机全局目录。

- [ ] **步骤 7：提交兼容验证和旧 Skill 删除**

```bash
git add global/skills/coordinating-cross-stack-sessions
git diff --cached --check
git diff --cached --name-status
git commit -m "refactor(协调): 移除旧跨端会话入口"
```

预期：提交只包含旧 Skill 的删除。

---

### 任务 7：完成最终评审与验证

**文件：**
- 评审：`global/skills/coordination-core/**`
- 评审：`global/skills/coordination-manual/**`
- 评审：`global/skills/coordination-leader/**`
- 评审：`global/src/**/AGENTS.md`
- 对照：`docs/superpowers/specs/2026-07-29-coordination-modes-design.md`

- [ ] **步骤 1：运行规格覆盖检查**

逐项映射以下需求到实际文件：

```text
共享协议唯一来源
手动 role/action/root/task 参数
task 可选和 current-task 解析
正式契约与不可变快照
角色独占写入目录
blocker/decision/handoff
任务和角色状态机
新会话恢复
Workflow 四模式预算兼容
Leader 累计 Agent 预算
禁止同一时刻混合执行
分层 AGENTS.md 与 coordinator 抽象
```

任何一项无法指向实际文件和章节时，停止完成声明并补齐缺失。

- [ ] **步骤 2：请求定向代码审查**

使用 `superpowers-zh:requesting-code-review`，审查重点：

- 模式切换是否存在角色抢占漏洞。
- 手动模式是否可能隐式派发 Agent 或虚构 Leader。
- Leader 是否可能绕过 Workflow 累计预算。
- 正式契约和快照是否形成两个可编辑事实源。
- 三个 Skill 是否重复定义共享协议。
- 角色规则是否存在越权或测试修改生产代码的入口。

发现问题时使用 `superpowers-zh:receiving-code-review` 验证并逐项修复，修复后重新运行受影响的场景。

- [ ] **步骤 3：运行最终验证命令**

```bash
git diff --check HEAD~5..HEAD
git log -5 --oneline --decorate
git status --short
find global/skills/coordination-core \
     global/skills/coordination-manual \
     global/skills/coordination-leader \
     global/src -maxdepth 3 -type f -print | sort
```

预期：

- 最近提交包含 core、manual、leader、角色规则和旧入口删除。
- 工作树只剩任务开始前已经存在的无关修改。
- 三个 Skill 文件和四份分层 `AGENTS.md` 均存在。

- [ ] **步骤 4：确认未同步全局目录**

```bash
test ! -e /Users/edy/.agents/skills/coordination-core
test ! -e /Users/edy/.agents/skills/coordination-manual
test ! -e /Users/edy/.agents/skills/coordination-leader
```

预期：全部退出码为 `0`。

- [ ] **步骤 5：使用 completion 验证清单**

调用 `superpowers-zh:verification-before-completion`，只依据本轮新鲜命令输出报告：

- 变更文件。
- Skill 和 Schema 验证证据。
- 手动、Leader、模式切换、契约漂移场景结果。
- 官方校验器是否因 `PyYAML` 阻塞。
- 未同步全局目录。
- 现有无关工作树修改和残余风险。

- [ ] **步骤 6：执行分支收尾**

调用 `superpowers-zh:finishing-a-development-branch`。本任务不创建或切换分支，因此只提供适用于当前分支的集成、保留或后续处理选项；不推送远端。
