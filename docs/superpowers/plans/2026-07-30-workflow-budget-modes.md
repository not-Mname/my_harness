# Workflow 流程预算模式实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 用自动、轻量、中量和重量 4 种 Superpowers 流程预算模式替代现有轻量、完整模式及 `Trivial / Standard / Complex` 分级，并同步到本机全局配置。

**架构：** `global/AGENTS.md` 只声明默认模式和共享策略入口；4 个模式 Skill 负责对话状态，`global/rules/workflow-policy.md` 是预算、路由、计数和异常处理的单一事实来源。自动模式逐任务临时路由，固定模式持续锁定；只有自动模式允许隐式调用。

**技术栈：** Markdown 规则、Codex Skill YAML 元数据、Shell 静态断言、Codex Skill `quick_validate.py`。

---

## 文件结构

- 创建 `global/rules/workflow-policy.md`：预算、自动路由、计数、优先级和异常处理。
- 创建 `global/skills/workflow-auto/`：自动模式状态与隐式入口。
- 修改 `global/skills/workflow-light/`：速度优先的固定模式入口。
- 创建 `global/skills/workflow-medium/`：平衡速度与质量的固定模式入口。
- 删除 `global/skills/workflow-full/`，创建 `global/skills/workflow-heavy/`：迁移完整流程语义。
- 修改 `global/AGENTS.md`：默认使用自动模式并加载共享策略。
- 按需修改 `global/rules/multi-agent-assistance.md`：删除旧分级和两档模式措辞。
- 同步对应文件到 `/Users/edy/.codex/` 和 `/Users/edy/.agents/skills/`；`global/root/` 不同步。

## 实施约束

- `global/AGENTS.md` 和若干规则、目录已有用户未提交修改；只能增量编辑，不恢复、覆盖或提交无关改动。
- 不创建 Worktree、分支或标签，不执行推送。
- 不保留 `$workflow-full` 或 `[模式：完整]` 兼容别名，不改写历史规格和历史计划。
- 每次提交前执行 `git diff --cached --check` 并核对暂存文件。

### 任务 1：建立共享流程策略

**文件：**
- 创建：`global/rules/workflow-policy.md`
- 参考：`docs/superpowers/specs/2026-07-30-workflow-budget-modes-design.md`

- [ ] **步骤 1：运行缺失断言**

运行：`test -f global/rules/workflow-policy.md`

预期：FAIL，退出码为 `1`。

- [ ] **步骤 2：写入模式状态、预算与裁剪规则**

使用 `apply_patch` 创建文件。必须逐项实现规格中的“模式状态”“流程预算”与“组件职责”，并包含以下冻结矩阵：

```markdown
| 维度 | 轻量级 | 中量级 | 重量级 |
|---|---|---|---|
| 可读写 Agent | 每任务累计最多 1 个 | 每任务累计最多 3 个 | Harness 不限制 |
| 只读 Agent | 每任务累计最多 3 个 | 每任务累计最多 5 个 | Harness 不限制 |
| 测试轮次 | 每任务最多 1 次 | 每任务最多 3 次 | Harness 不限制 |
| 澄清轮次 | 每任务最多 3 次 | 每任务最多 5 次 | Harness 不限制 |
```

轻量级必须压缩 brainstorming，默认不生成持久化规格、计划或评审且不强制 TDD；中量级合并设计往返并保留关键规格、计划、测试和评审；重量级完整执行所有适用流程。

- [ ] **步骤 3：写入计数、自动路由和异常处理**

逐条转写规格中的全部路由条件，不得缩写为“按复杂度判断”。写入以下计数与优先级契约：

```markdown
- Agent 配额按单任务累计派发量计算；完成、失败、中断和返工派发均占用额度。
- 子 Agent 不得继续派发；只读 Agent 不得升级为可读写 Agent。
- 修复后重新运行计为新测试轮次。
- lint、类型检查、构建和测试框架计入测试轮次；git diff、只读搜索和纯静态检查不计入。
- 强制 HITL 不计入普通澄清轮次，也不得因配额耗尽而跳过。

1. 系统、平台、安全、数据、外部副作用和强制 HITL 规则。
2. 用户当前任务的明确执行要求或显式指定 Skill。
3. 用户手动锁定的固定模式。
4. 自动模式临时档位。
5. Superpowers Skill 默认完整流程。
6. Harness 一般效率偏好。
```

异常处理明确：配额在开始时规划；自动模式预算不足时升级且不降级；固定模式重新规划但不自动升级；只有不可绕过底线冲突时进入 HITL。

- [ ] **步骤 4：验证并提交共享策略**

运行：

```bash
rg -q '^## 模式预算$' global/rules/workflow-policy.md
rg -q '^## 自动路由$' global/rules/workflow-policy.md
rg -q '每任务累计最多 1 个' global/rules/workflow-policy.md
rg -q '每任务累计最多 5 个' global/rules/workflow-policy.md
rg -q '不得自动降级' global/rules/workflow-policy.md
! rg -n 'Trivial|Standard|Complex' global/rules/workflow-policy.md
git add global/rules/workflow-policy.md
git diff --cached --check
git diff --cached --name-status
git commit -m "feat(流程): 添加共享流程预算策略"
```

预期：静态断言通过，暂存列表和提交只包含共享策略文件。

### 任务 2：实现 4 个模式 Skill

**文件：**
- 创建：`global/skills/workflow-auto/SKILL.md`、`agents/openai.yaml`
- 修改：`global/skills/workflow-light/SKILL.md`、`agents/openai.yaml`
- 创建：`global/skills/workflow-medium/SKILL.md`、`agents/openai.yaml`
- 删除：`global/skills/workflow-full/SKILL.md`、`agents/openai.yaml`
- 创建：`global/skills/workflow-heavy/SKILL.md`、`agents/openai.yaml`

- [ ] **步骤 1：运行结构失败断言**

运行：`test -f global/skills/workflow-auto/SKILL.md`

预期：FAIL，退出码为 `1`。

- [ ] **步骤 2：创建自动模式 Skill**

`SKILL.md` 写入：

```markdown
---
name: workflow-auto
description: Use by default for each new Codex conversation, or when 用户显式调用 $workflow-auto，要求恢复逐任务自动判断流程预算
---

# 持续自动判断流程

开始任务前必须读取 `~/.codex/rules/workflow-policy.md`。

1. 新对话默认进入自动模式；每个任务独立判断轻量级、中量级或重量级。
2. 输出 `[自动模式 -> 档位]` 和一句核心理由后立即执行，不请求用户批准。
3. 本次判定不改变持续模式；下一个任务重新判断。
4. 任务中途发现新风险时可以升级并告知，不得自动降级。
5. 显式调用不含任务正文时，只确认已恢复持续自动判断流程。
6. 最新显式模式切换优先，任务中途切换只影响未完成步骤。
```

元数据写入：

```yaml
interface:
  display_name: "持续自动判断流程"
  short_description: "为每个任务自动选择轻量、中量或重量流程预算"
  default_prompt: "使用 $workflow-auto 将当前对话恢复为逐任务自动判断流程。"
policy:
  allow_implicit_invocation: true
```

- [ ] **步骤 3：实现 3 个固定模式 Skill**

3 个固定模式都必须读取共享策略；显式调用后持续锁定；携带正文时立即执行；单独调用时只确认；不自动升降级；仅在不可绕过底线冲突时进入 HITL。

使用以下冻结元数据：

```text
workflow-light  / 持续轻量级流程 / 速度优先 / allow_implicit_invocation: false
workflow-medium / 持续中量级流程 / 速度与质量平衡 / allow_implicit_invocation: false
workflow-heavy  / 持续重量级流程 / 完整执行适用 Superpowers / allow_implicit_invocation: false
```

- [ ] **步骤 4：删除旧完整模式**

使用 `apply_patch` 删除 `global/skills/workflow-full/SKILL.md` 和 `global/skills/workflow-full/agents/openai.yaml`，不得保留兼容别名。

- [ ] **步骤 5：校验并提交模式 Skill**

运行：

```bash
for mode in auto light medium heavy; do
  python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py "global/skills/workflow-${mode}"
done
rg -q 'allow_implicit_invocation: true' global/skills/workflow-auto/agents/openai.yaml
for mode in light medium heavy; do
  rg -q 'allow_implicit_invocation: false' "global/skills/workflow-${mode}/agents/openai.yaml"
done
test ! -e global/skills/workflow-full
! rg -n 'Trivial|Standard|Complex|\$workflow-full|模式：完整' global/skills/workflow-{auto,light,medium,heavy}
git add global/skills/workflow-{auto,light,medium,full,heavy}
git diff --cached --check
git diff --cached --name-status
git commit -m "feat(流程): 扩展四档流程预算模式"
```

预期：4 个 Skill 均输出 `Skill is valid!`；只有自动模式允许隐式调用；旧目录不存在；提交只包含模式目录。

### 任务 3：接入全局入口和多 Agent 规则

**文件：**
- 修改：`global/AGENTS.md`
- 按需修改：`global/rules/multi-agent-assistance.md`

- [ ] **步骤 1：运行默认入口失败断言**

运行：

```bash
rg -q '默认使用 `\$workflow-auto\`' global/AGENTS.md
```

预期：FAIL；当前文件仍声明 `$workflow-full`。

- [ ] **步骤 2：增量修改全局入口**

只替换流程模式章节，不改写该文件已有重组内容：

```markdown
## 流程模式

- 默认使用 `$workflow-auto`。
- 出现任一流程模式 Skill 或模式标签，或任务需要开发流程时，须先读取 `~/.codex/rules/workflow-policy.md`。
```

- [ ] **步骤 3：定向清理多 Agent 规则旧语义**

运行：

```bash
rg -n 'Trivial|Standard|Complex|完整模式|轻量模式' global/rules/multi-agent-assistance.md
```

删除“服从全局流程分级”；派发数量引用当前模式预算；自动、中量和重量模式在设计、计划、任务清单或失败分组变化后重新判断并行条件。不得重构其他并行准入、拆分或汇合逻辑。

- [ ] **步骤 4：验证有效规则与隔离边界**

运行：

```bash
rg -q '默认使用 `\$workflow-auto\`' global/AGENTS.md
rg -q '~/.codex/rules/workflow-policy.md' global/AGENTS.md
! rg -n 'Trivial|Standard|Complex|\$workflow-full|模式：完整' \
  global/AGENTS.md global/rules/workflow-policy.md global/rules/multi-agent-assistance.md \
  global/skills/workflow-{auto,light,medium,heavy}
git diff --check
git diff -- global/AGENTS.md global/rules/multi-agent-assistance.md
git status --short
```

预期：有效规则无旧术语，diff 无空白错误，用户此前同文件改动仍保留。只有能明确隔离本任务补丁时才暂存，否则保持未暂存并在交付中说明。

### 任务 4：整体验证并同步本机配置

**仓库源：**
- `global/AGENTS.md`
- `global/rules/workflow-policy.md`
- `global/skills/workflow-auto/`
- `global/skills/workflow-light/`
- `global/skills/workflow-medium/`
- `global/skills/workflow-heavy/`

**本机目标：**
- `/Users/edy/.codex/AGENTS.md`
- `/Users/edy/.codex/rules/workflow-policy.md`
- `/Users/edy/.agents/skills/workflow-auto/`
- `/Users/edy/.agents/skills/workflow-light/`
- `/Users/edy/.agents/skills/workflow-medium/`
- `/Users/edy/.agents/skills/workflow-heavy/`
- 删除 `/Users/edy/.agents/skills/workflow-full/` 中与旧 Skill 对应的配置文件。

- [ ] **步骤 1：执行仓库最终验证**

运行：

```bash
git diff --check
for mode in auto light medium heavy; do
  python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py "global/skills/workflow-${mode}"
done
rg -q '默认使用 `\$workflow-auto\`' global/AGENTS.md
rg -q '^## 自动路由$' global/rules/workflow-policy.md
test ! -e global/skills/workflow-full
```

预期：diff 检查通过，4 个 Skill 有效，默认入口和自动路由存在，旧仓库 Skill 不存在。

- [ ] **步骤 2：复核路由场景矩阵**

确认共享策略可唯一推出：

```text
目标明确的只读解释 -> 轻量级
常规多文件功能且边界清晰 -> 中量级
架构或公共契约变更 -> 重量级
安全、数据迁移或不可逆副作用 -> 重量级
自动轻量任务发现第二轮必要测试 -> 升级中量级
手动轻量任务范围扩大 -> 保持轻量并重新规划
```

- [ ] **步骤 3：同步前核对精确目标**

运行：

```bash
test -d /Users/edy/.codex/rules
test -d /Users/edy/.agents/skills
find /Users/edy/.agents/skills -maxdepth 2 -path '*/workflow-*/*' -type f -print | sort
```

预期：父目录存在；发现仓库源之外的未知文件时不删除，先报告。

- [ ] **步骤 4：使用 `apply_patch` 同步文本文件**

同步映射：

```text
global/AGENTS.md -> /Users/edy/.codex/AGENTS.md
global/rules/workflow-policy.md -> /Users/edy/.codex/rules/workflow-policy.md
global/skills/workflow-auto/* -> /Users/edy/.agents/skills/workflow-auto/*
global/skills/workflow-light/* -> /Users/edy/.agents/skills/workflow-light/*
global/skills/workflow-medium/* -> /Users/edy/.agents/skills/workflow-medium/*
global/skills/workflow-heavy/* -> /Users/edy/.agents/skills/workflow-heavy/*
```

删除旧全局 `workflow-full` 的 `SKILL.md` 和已知元数据文件；`.DS_Store` 不属于配置，不同步也不主动删除。

- [ ] **步骤 5：比较并校验本机安装内容**

运行：

```bash
cmp global/AGENTS.md /Users/edy/.codex/AGENTS.md
cmp global/rules/workflow-policy.md /Users/edy/.codex/rules/workflow-policy.md
for mode in auto light medium heavy; do
  cmp "global/skills/workflow-${mode}/SKILL.md" "/Users/edy/.agents/skills/workflow-${mode}/SKILL.md"
  cmp "global/skills/workflow-${mode}/agents/openai.yaml" "/Users/edy/.agents/skills/workflow-${mode}/agents/openai.yaml"
  python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py "/Users/edy/.agents/skills/workflow-${mode}"
done
test ! -e /Users/edy/.agents/skills/workflow-full/SKILL.md
```

预期：所有 `cmp` 退出码为 `0`，4 个本机 Skill 有效，旧全局 Skill 主文件不存在。

- [ ] **步骤 6：最终交付检查**

运行：

```bash
git diff --check
git status --short
git log -4 --oneline --decorate
```

预期：没有本任务引入的空白错误；状态清楚区分本任务与用户已有修改。最终回答报告仓库变更、提交、验证、全局同步、保留的用户改动和未暂存目标文件。
