# Workflow 模式规则整合设计

## 目标

将流程模式规则从共享文件 `global/rules/workflow-modes.md` 拆分并内嵌到两个显式调用 Skill，使 `workflow-full` 与 `workflow-light` 各自包含完整的模式行为，不再依赖共享流程模式文件。

## 范围

- 将 `global/AGENTS.md` 的“流程模式”章节缩减为默认使用 `$workflow-full`。
- 保留 `global/AGENTS.md` 的其他章节，不改动其行为。
- 将完整模式规则并入 `global/skills/workflow-full/SKILL.md`。
- 将轻量模式规则并入 `global/skills/workflow-light/SKILL.md`。
- 删除 `global/rules/workflow-modes.md`。
- 保留 `global/rules/codegraph-navigation.md`、`editor-mcp-checkpoint.md`、`git-development.md` 与 `parallel-execution.md`。
- 不修改 `global/root/`。
- 本次只修改仓库规范源，不同步到 `~/.codex` 或 `~/.agents`。

## 结构设计

### 全局入口

`global/AGENTS.md` 的流程模式章节只声明新对话默认使用 `$workflow-full`。显式模式切换及具体执行规则由对应 Skill 自身负责。

### workflow-full

`global/skills/workflow-full/SKILL.md` 保留持续切换语义，并内嵌以下内容：

- 显式调用、携带任务、单独调用、最新切换优先和新对话复位规则。
- 完整流程对纯问答、功能修改、Bug 修复、计划与并行门禁的处理方式。
- Trivial、Standard、Complex 流程分级。
- HITL、Editor MCP、CodeGraph、Git、外部副作用和高风险验证等始终生效规则。

### workflow-light

`global/skills/workflow-light/SKILL.md` 保留持续切换语义，并内嵌以下内容：

- 显式调用、携带任务、单独调用、最新切换优先和新对话复位规则。
- 轻量流程的最少读取、默认跳过项、有限并行和最小充分验证要求。
- 无法安全轻量完成时必须进入 HITL，不得静默升级。
- Trivial、Standard、Complex 流程分级。
- 与完整模式相同的始终生效安全规则。

两个 Skill 允许重复必要的公共内容，以换取自包含性和独立可读性。

## 引用与兼容性

- 两个 Skill 删除“读取 `~/.codex/rules/workflow-modes.md`”步骤。
- 其余四个规则文件继续由 `global/AGENTS.md` 的按需规则引用。
- 两个 Skill 的 `name`、`description` 与 `agents/openai.yaml` 元数据保持不变。
- `[模式：轻量]`、`[模式：完整]`、`$workflow-light` 与 `$workflow-full` 的持续切换语义保持不变。

## 验证

1. 确认仓库中不存在对 `workflow-modes.md` 的有效引用。
2. 确认 `global/rules/workflow-modes.md` 已删除，其余四个规则文件仍存在且内容未被本任务修改。
3. 使用 Codex Skill 快速校验脚本验证两个 Skill。
4. 检查 `global/AGENTS.md` 仅缩减流程模式章节，其他章节保持一致。
5. 检查 `global/root/` 无变更。
6. 检查 diff，确保不包含用户已有的 `global/rules/git-development.md` 修改。

## 风险

- 公共规则在两个 Skill 中重复，后续修改必须同步更新两处。
- 在仓库规范源同步到本机前，本机已安装 Skill 仍会继续读取旧的 `~/.codex/rules/workflow-modes.md`；同步属于后续独立操作。
