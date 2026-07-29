# Workflow 模式规则整合实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将共享的流程模式规则拆分进两个自包含 Skill，并把全局入口缩减为默认使用 `$workflow-full`。

**Architecture:** `global/AGENTS.md` 只负责声明默认模式；`workflow-full` 与 `workflow-light` 分别承载自身的切换语义、执行流程、分级和始终生效规则。删除共享 `workflow-modes.md`，保留其余规则文件及 `global/root/`。

**Tech Stack:** Markdown、Codex Skill YAML frontmatter、Shell 契约检查、Codex `quick_validate.py`

## Global Constraints

- 只处理流程模式内容。
- 保留 `global/AGENTS.md` 中除“流程模式”章节外的全部内容。
- 保留 `global/rules/codegraph-navigation.md`、`editor-mcp-checkpoint.md`、`git-development.md` 与 `parallel-execution.md`。
- 不修改 `global/root/`。
- 不同步到 `~/.codex` 或 `~/.agents`。
- 不覆盖或提交用户已有的 `global/rules/git-development.md` 修改。
- 不修改两个 `agents/openai.yaml` 文件。

---

### Task 1: 整合流程模式规则

**Files:**
- Modify: `global/AGENTS.md`
- Modify: `global/skills/workflow-full/SKILL.md`
- Modify: `global/skills/workflow-light/SKILL.md`
- Delete: `global/rules/workflow-modes.md`
- Preserve: `global/rules/codegraph-navigation.md`
- Preserve: `global/rules/editor-mcp-checkpoint.md`
- Preserve: `global/rules/git-development.md`
- Preserve: `global/rules/parallel-execution.md`
- Preserve: `global/root/**`

**Interfaces:**
- Consumes: `$workflow-full`、`$workflow-light`、`[模式：完整]` 与 `[模式：轻量]` 的现有持续切换契约。
- Produces: 两个不依赖 `~/.codex/rules/workflow-modes.md` 的自包含 Skill；新对话由 `global/AGENTS.md` 默认选择 `$workflow-full`。

- [ ] **Step 1: 运行变更前契约检查并确认失败**

Run:

```bash
bash -c '
set -e
test ! -e global/rules/workflow-modes.md
test "$(sed -n "/^## 流程模式$/,/^## 按需规则$/p" global/AGENTS.md)" = "## 流程模式

默认使用 \`\$workflow-full\`。

## 按需规则"
! rg -n "workflow-modes\.md" global/skills/workflow-full/SKILL.md global/skills/workflow-light/SKILL.md
'
```

Expected: FAIL，因为 `global/rules/workflow-modes.md` 仍存在，且两个 Skill 仍引用该文件。

- [ ] **Step 2: 缩减全局流程模式入口**

将 `global/AGENTS.md` 中从 `## 流程模式` 到 `## 按需规则` 之前的内容替换为：

```markdown
## 流程模式

默认使用 `$workflow-full`。
```

不得修改该文件的其他章节。

- [ ] **Step 3: 将完整流程规则写入 workflow-full**

将 `global/skills/workflow-full/SKILL.md` 完整替换为：

```markdown
---
name: workflow-full
description: Use when 用户显式调用 $workflow-full，要求将当前 Codex 对话持续切换为完整流程
---

# 持续完整流程

将当前对话切换为持续完整模式，而不是只覆盖当前任务。

## 模式切换

1. 用户显式调用 `$workflow-full` 或使用 `[模式：完整]` 时，将当前对话的持续模式记录为完整，直到用户显式调用 `$workflow-light`、使用 `[模式：轻量]`，或开始新对话。
2. 调用消息包含任务正文时，切换后立即按完整流程执行该任务；后续任务继续使用完整流程。
3. 调用消息没有任务正文时，只确认“已切换为持续完整流程”，随后停止。
4. 最新显式切换优先；任务中途切换只影响未完成步骤，不撤销已有修改、验证或授权。
5. 不把本次调用解释为临时覆盖，不恢复调用前的模式；`[完整]` 与 `[轻量]` 不视为模式指令。
6. 完整表示执行全部适用流程，不机械调用无关 Skill。

## 完整流程

- 纯问答、翻译、只读查询和无逻辑微调直接处理。
- 创意、功能或行为修改执行 `brainstorming -> writing-plans -> 执行流程`。
- Bug 或异常执行 `systematic-debugging -> test-driven-development -> 实现 -> verification-before-completion`。
- 计划、任务清单或失败分组变化后重新执行并行门禁；无依赖任务并行，带依赖任务按依赖层执行。
- SDD 只负责串行实施与逐项评审；完成开发后执行适用测试、代码评审、整体验证和分支收尾。

## 流程分级

- **Trivial**：单文件、20 行以内、无逻辑变化，或单文件阅读/一次 CodeGraph 可解决；直接处理、检查 diff，默认不派发。
- **Standard**：常规功能或多文件协作；最多 3 个业务子任务。只读 Micro-Agent 计入业务子任务；评审或修复 Agent 不另算业务子任务，同时活动的子 Agent 总数不得超过 3。
- **Complex**：架构调整、跨模块重构或安全关键；可随独立工作扩展业务子任务，同时活动数量仍受平台和并行规则限制。

测试按风险选择最小充分集合；共享基础设施、数据迁移、安全及关键流程不得省略必要验证。预计超过 40 分钟时拆成可独立验收且每项以 40 分钟内完成为目标的任务。

## 始终生效

不得绕过 HITL、假设协议、Editor MCP、场景和资源确认、Git 历史改写确认与禁止推送、能力恢复、CodeGraph、数据和外部副作用检查、必要高风险验证、更高优先级指令及用户明确指定的 Skill。
```

- [ ] **Step 4: 将轻量流程规则写入 workflow-light**

将 `global/skills/workflow-light/SKILL.md` 完整替换为：

```markdown
---
name: workflow-light
description: Use when 用户显式调用 $workflow-light，要求将当前 Codex 对话持续切换为轻量流程
---

# 持续轻量流程

将当前对话切换为持续轻量模式，而不是只覆盖当前任务。

## 模式切换

1. 用户显式调用 `$workflow-light` 或使用 `[模式：轻量]` 时，将当前对话的持续模式记录为轻量，直到用户显式调用 `$workflow-full`、使用 `[模式：完整]`，或开始新对话。
2. 调用消息包含任务正文时，切换后立即按轻量流程执行该任务；后续任务继续使用轻量流程。
3. 调用消息没有任务正文时，只确认“已切换为持续轻量流程”，随后停止。
4. 最新显式切换优先；任务中途切换只影响未完成步骤，不撤销已有修改、验证或授权。
5. 不把本次调用解释为临时覆盖，不恢复调用前的模式；`[完整]` 与 `[轻量]` 不视为模式指令。

## 轻量流程

用户选择轻量模式，视为明确要求跳过 Superpowers 的非安全过程：

1. 识别范围、风险和必须加载的按需规则，读取完成任务所需的最少文件后直接处理。
2. 默认不生成规格、实施计划、Worktree、TDD 循环或代码评审，也不派发 Agent。
3. 两个以上明显独立的只读问题只有在确实降低耗时时才可按并行规则派发。
4. 使用最小充分验证，优先 diff、静态检查、定向测试或最短复现；20 行以内且无逻辑变化时仅检查 diff。
5. 最终回答仅报告变更、验证、风险和未完成项。

无法在不降低安全性或必要验证的前提下轻量完成时，不得静默升级；须通过 HITL 请求切换完整模式或缩小范围。

## 流程分级

- **Trivial**：单文件、20 行以内、无逻辑变化，或单文件阅读/一次 CodeGraph 可解决；直接处理、检查 diff，默认不派发。
- **Standard**：常规功能或多文件协作；最多 3 个业务子任务。只读 Micro-Agent 计入业务子任务；评审或修复 Agent 不另算业务子任务，同时活动的子 Agent 总数不得超过 3。
- **Complex**：架构调整、跨模块重构或安全关键；可随独立工作扩展业务子任务，同时活动数量仍受平台和并行规则限制。

测试按风险选择最小充分集合；共享基础设施、数据迁移、安全及关键流程不得省略必要验证。预计超过 40 分钟时拆成可独立验收且每项以 40 分钟内完成为目标的任务。

## 始终生效

不得绕过 HITL、假设协议、Editor MCP、场景和资源确认、Git 历史改写确认与禁止推送、能力恢复、CodeGraph、数据和外部副作用检查、必要高风险验证、更高优先级指令及用户明确指定的 Skill。
```

- [ ] **Step 5: 删除共享流程模式文件**

删除：

```text
global/rules/workflow-modes.md
```

- [ ] **Step 6: 运行契约检查并确认通过**

Run:

```bash
bash -c '
set -e
test ! -e global/rules/workflow-modes.md
test "$(sed -n "/^## 流程模式$/,/^## 按需规则$/p" global/AGENTS.md)" = "## 流程模式

默认使用 \`\$workflow-full\`。

## 按需规则"
! rg -n "workflow-modes\.md" global/skills/workflow-full/SKILL.md global/skills/workflow-light/SKILL.md
rg -q "^## 完整流程$" global/skills/workflow-full/SKILL.md
rg -q "^## 轻量流程$" global/skills/workflow-light/SKILL.md
rg -q "^## 流程分级$" global/skills/workflow-full/SKILL.md
rg -q "^## 流程分级$" global/skills/workflow-light/SKILL.md
rg -q "^## 始终生效$" global/skills/workflow-full/SKILL.md
rg -q "^## 始终生效$" global/skills/workflow-light/SKILL.md
'
```

Expected: PASS，退出码为 0，无输出。

- [ ] **Step 7: 使用官方脚本校验两个 Skill**

Run:

```bash
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py global/skills/workflow-full
python3 /Users/edy/.codex/skills/.system/skill-creator/scripts/quick_validate.py global/skills/workflow-light
```

Expected: 两次均输出 `Skill is valid!`，退出码均为 0。

- [ ] **Step 8: 验证保留文件和修改边界**

Run:

```bash
shasum -a 256 global/rules/codegraph-navigation.md global/rules/editor-mcp-checkpoint.md global/rules/git-development.md global/rules/parallel-execution.md
find global/root -type f -exec shasum -a 256 {} \; | sort
git diff -- global/AGENTS.md global/skills/workflow-full/SKILL.md global/skills/workflow-light/SKILL.md global/rules/workflow-modes.md
git status --short
```

Expected rule hashes:

```text
41a8c6274f15e808a6896db1aec0769de2603d078aa12f543bc9ab1e7a1a16fc  global/rules/codegraph-navigation.md
f229b8ce8b0e7fe95bf026d792f58875ecc268962eac58fb03244c1606e97757  global/rules/editor-mcp-checkpoint.md
10e44dab12562921f5d8f22b326c74a3d143fa6b53eb33468d81fb99d7d8eda0  global/rules/git-development.md
36924168b37fb88d7382aea9853032cf565e38c10be2bb66f5dadb832517f665  global/rules/parallel-execution.md
```

Expected root hashes:

```text
098a1da34266ba48df0f079b1e2434235c3a016458a0f9d9dac5f67a1d357a00  global/root/AGENTS.md
3fee03b60eb6d8561d3817c9d7644dba58c3e527759f8883e646c81c63cf0794  global/root/client/AGENTS.md
eca9d34b05b87b79e7b4c58695a204d1289c3b67d9739ac3fd245e08a3fc60c2  global/root/server/AGENTS.md
```

Expected: diff 仅包含 3 个修改文件和 1 个删除文件；`git status --short` 还会显示用户已有的 ` M global/rules/git-development.md`，但该文件内容和上方基线哈希一致。

- [ ] **Step 9: 精确暂存、复核并提交实现**

Run:

```bash
git add global/AGENTS.md global/skills/workflow-full/SKILL.md global/skills/workflow-light/SKILL.md global/rules/workflow-modes.md
git diff --cached --check
git diff --cached --stat
git status --short
git commit -m "整合流程模式规则"
```

Expected: 暂存区只包含 3 个修改文件和 1 个删除文件；`global/rules/git-development.md` 保持未暂存；提交成功。
