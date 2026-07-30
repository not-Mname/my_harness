# Workflow 策略歧义消除实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 完善 Workflow 策略中的任务边界、验证选择、返工计数、显式 Skill 放宽和不可绕过底线，并同步本机全局规则。

**架构：** 所有行为继续集中在 `global/rules/workflow-policy.md`，4 个模式 Skill 不复制策略。修改采用定向章节扩展，不拆分 rule；仓库验证通过后镜像到 `/Users/edy/.codex/rules/workflow-policy.md`。

**技术栈：** Markdown 规则、Shell 静态断言、Git diff、`cmp`。

---

## 文件结构

- 修改 `global/rules/workflow-policy.md`：补全术语、生命周期、验证选择、计数、冲突与底线。
- 更新 `/Users/edy/.codex/rules/workflow-policy.md`：逐字节同步仓库规则。
- 不修改 4 个模式 Skill。
- 不读取、修改或提交 `global/skills/coordination-leader/`。

### 任务 1：完善并同步 Workflow 策略

**文件：**
- 修改：`global/rules/workflow-policy.md`
- 同步：`/Users/edy/.codex/rules/workflow-policy.md`
- 参考：`docs/superpowers/specs/2026-07-30-workflow-policy-ambiguity-hardening-design.md`

- [ ] **步骤 1：运行缺失行为断言**

运行：

```bash
rg -q '^## 术语定义$' global/rules/workflow-policy.md
rg -q '^## 不可绕过底线$' global/rules/workflow-policy.md
rg -q '本模式不额外限制' global/rules/workflow-policy.md
```

预期：第一条即失败，退出码为 `1`。

- [ ] **步骤 2：新增术语定义**

在“使用方式”后写入以下完整契约：

```markdown
## 术语定义

- **任务**：单次用户提示要求的完整交付单元，共享一次模式判定和一组 Agent、测试及澄清预算。
- **子任务**：Agent 为完成任务而拆出的内部工作项，不重置任务预算。
- 一次提示包含多个独立子目标时，默认仍属于一个任务。只有用户明确要求分别交付、分别验收，或 Agent 通过 HITL 获得拆分确认后，才建立多个独立任务并分别计数。
- **对话生命周期**：手动固定模式持续到用户再次切换或当前对话结束；新对话默认进入自动模式。
- **返工派发**：前序 Agent 输出不满足要求而进行的任何修正性派发，无论对象是同一还是新的子 Agent。主 Agent 自行完成的修正不计入派发额度。
```

- [ ] **步骤 3：统一模式术语并补全轻量级验证选择**

将预算矩阵和重量级说明中的 `Harness 不限制` 改为“本模式不额外限制”，并写明“本模式”不突破平台限制或系统规则。

在轻量级验证规则后加入：

```markdown
验证选择顺序：

1. 优先选择能覆盖核心逻辑、主要边界和主要失败模式的现有端到端或集成测试。
2. 不具备上述测试时，选择覆盖本次修改最关键路径的定向测试。
3. 仍无合适测试时，选择最接近用户可观察结果的构建、类型检查或 lint。
4. 不得为了扩大覆盖而临时拼接多个原本独立的测试目标。
```

- [ ] **步骤 4：收紧显式 Skill 放宽规则**

将规则优先级后的说明替换为：

```markdown
用户显式要求的 Skill 必须执行。必要步骤超过当前档位配额时，只放宽完成该 Skill 硬性步骤所需的最低配额并简短声明；超出原上限的使用量仍须记录，不改变对话的持续模式。

例如，显式要求完整 TDD 时，红、绿、重构循环所必需的测试轮次，以及直接参与该循环的 Agent 可以超过原上限；非核心文档、额外探索和一般性评审仍受原模式预算限制。不得借显式 Skill 将整个任务无条件提升为重量级。
```

- [ ] **步骤 5：新增不可绕过底线和固定模式冲突闭环**

在规则优先级后新增设计规格中的完整非穷尽底线清单。异常处理中明确：

```text
当前[模式]与[具体不可绕过规则]冲突。可选处理：切换至[所需模式]、缩小任务范围或停止。
```

只能接受切换模式、缩小范围或停止；不得提供忽略底线选项。用户明确选择前，不得继续受限制的修改或副作用。

- [ ] **步骤 6：运行策略验证**

运行：

```bash
rg -q '^## 术语定义$' global/rules/workflow-policy.md
rg -q '^## 不可绕过底线$' global/rules/workflow-policy.md
rg -q '默认仍属于一个任务' global/rules/workflow-policy.md
rg -q '主 Agent 自行完成的修正不计入' global/rules/workflow-policy.md
rg -q '本模式不额外限制' global/rules/workflow-policy.md
rg -q '端到端或集成测试' global/rules/workflow-policy.md
rg -q '硬性步骤所需的最低配额' global/rules/workflow-policy.md
rg -q '缩小任务范围或停止' global/rules/workflow-policy.md
! rg -n 'Harness 不限制|授权忽略|忽略底线' global/rules/workflow-policy.md
git diff --check -- global/rules/workflow-policy.md
```

预期：全部断言通过，负向扫描无输出，diff 检查退出码为 `0`。

- [ ] **步骤 7：提交仓库策略**

运行：

```bash
git add global/rules/workflow-policy.md
git diff --cached --check
git diff --cached --name-status
git commit -m "docs(流程): 完善流程预算策略边界"
```

预期：暂存和提交只包含 `global/rules/workflow-policy.md`。

- [ ] **步骤 8：同步并验证本机全局策略**

将仓库策略机械同步到 `/Users/edy/.codex/rules/workflow-policy.md`，然后运行：

```bash
cmp global/rules/workflow-policy.md /Users/edy/.codex/rules/workflow-policy.md
git diff --check
git status --short
```

预期：`cmp` 与 diff 检查退出码为 `0`；工作区仅保留用户原有的未跟踪 `global/skills/coordination-leader/`。
