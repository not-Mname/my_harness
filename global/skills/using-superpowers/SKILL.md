---
name: using-superpowers
description: 在任何任务开始时使用——先按 Workflow 档位确定流程预算，再按档位裁剪表路由技能；本技能取代插件同名技能，是技能调用的唯一入口
---

<SUBAGENT-STOP>
如果你是作为子智能体被分派来执行特定任务的，忽略此技能。
</SUBAGENT-STOP>

# Harness 技能路由

本技能是本项目（my_harness）对插件 `superpowers-zh:using-superpowers` 的定制版，取代插件同名技能。插件升级或未来与 superpowers 解耦时，只改本文件与 `global/skills/` 下的裁剪版技能。

## 第一步：确定档位

开始任何任务前，先读取 `~/.codex/rules/workflow-policy.md`：

1. 新对话默认进入自动模式，按任务独立判定 `轻量级 | 中量级 | 重量级`，输出 `[自动模式 -> 档位]` 和一句核心理由后立即执行，不请求用户批准。
2. 出现流程模式 Skill 或模式标签（`$workflow-auto` / `$workflow-light` / `$workflow-medium` / `$workflow-heavy`）时，按对应模式锁定；模式指令携带任务正文时先切换再执行。
3. 档位决定本任务允许的 Agent、测试、澄清配额与流程裁剪方式；配额不足时按策略升级或重新规划，不得静默省略验证。

## 技能调用规则

**只有当档位裁剪表命中时才调用对应技能。** 插件原版的“哪怕 1% 可能适用就必须调用”规则在本 harness 中不适用——档位未命中的任务直接执行，不调用被裁剪的技能，也不以“先调用看看”绕过。

### 档位裁剪表

| superpowers 技能 | 轻量级 | 中量级 | 重量级 |
|---|---|---|---|
| brainstorming | 压缩：必要澄清 + 简短方案确认，不写 spec | 合并澄清 + 关键设计确认，spec 可选 | 完整流程 + spec |
| systematic-debugging | 最小根因确认 | 流程化根因分析 | 完整流程 |
| test-driven-development | 不强制；唯一测试轮用于最高信息量验证 | 高风险逻辑优先 TDD，其余按风险选择 | 完整 TDD |
| writing-plans | 不生成计划，只维护简短任务清单 | 多步骤任务生成可压缩计划 | 完整计划 |
| executing-plans | — | 按需 | 按需 |
| subagent-driven-development | — | 按需，串行实现 + 评审 | 按需 |
| dispatching-parallel-agents | 一般不派发；受配额与独立条件约束 | 满足独立条件才并行 | 满足独立条件才并行 |
| using-git-worktrees | 默认禁止，须 HITL 授权 | 须 HITL 授权 | 须 HITL 授权 |
| verification-before-completion | 精简验证 | 标准验证 | 完整验证 |
| requesting-code-review / receiving-code-review | 不强制评审 | 重要修改定向评审 | 全面评审 |
| finishing-a-development-branch | 直接汇报 | 常规收尾 | 完整收尾 |
| chinese-* 系列 | 按需叠加 | 按需叠加 | 按需叠加 |

裁剪表中的技能优先使用 `global/skills/` 下的 harness 裁剪版；完整操作细节可参考插件原版，冲突时以裁剪版为准。

### Harness 专属技能

- 流程模式：`workflow-auto` / `workflow-light` / `workflow-medium` / `workflow-heavy`（模式切换与预算）
- 多 Agent 协作：派发前必读 `~/.codex/rules/multi-agent-assistance.md`；协调类任务使用 `coordination-core` / `coordination-leader` / `coordination-manual`
- 代码注释：`writing-contextual-code-comments`
- 人工确认与假设协议：见 `global/AGENTS.md` 与本文件下方优先级

## 规则优先级

1. 系统、平台、安全、数据、外部副作用和强制 HITL 规则（含 `global/AGENTS.md` 的 HITL Checkpoint `[CP-n]` 与假设协议）。
2. 用户在当前任务中明确提出的执行要求或显式指定的 Skill。
3. 用户手动锁定的固定模式。
4. 自动模式为当前任务选择的临时档位。
5. Superpowers Skill 自身的默认完整流程（被本表裁剪后按档位执行）。
6. 一般效率偏好。

用户显式要求的 Skill 必须执行；必要步骤超过当前档位配额时，只放宽完成该 Skill 硬性步骤所需的最低配额并简短声明，不把整个任务无条件提升为重量级。

## 澄清与假设

- 澄清轮次计入档位配额（轻量 3 轮 / 中量 5 轮 / 重量不额外限制）；合并相关问题，避免逐项往返。
- 低影响行业惯例、既有模式或无业务影响细节可标记 `[ASSUMED]` 并附依据，直接执行。
- 需求目标、数据结构、API 契约、交互流程、性能指标及场景、资源、项目设置、数据、安全或外部副作用不得假设，须进入 HITL Checkpoint。
- 强制 HITL（Git 历史改写、Editor MCP、支付/发布、数据迁移等）不计入普通澄清轮次，也不得因配额耗尽而跳过。

## 平台适配（Codex）

| 插件技能引用 | Codex 等价工具 |
|---|---|
| Task（派发子 agent） | `spawn_agent` |
| 多个 Task 并行 | 多个 `spawn_agent` 调用 |
| Task 返回 | `wait_agent` |
| Task 自动完成 | `close_agent` 释放槽位 |
| TodoWrite | `update_plan` |
| Read / Write / Edit / Bash | 原生文件与 shell 工具 |

多 Agent 能力受 `~/.codex/rules/multi-agent-assistance.md` 与档位派发配额约束，不以工具可用性绕过规则。

## 解耦声明

本路由表即 harness 对 superpowers 技能的依赖清单。未来替换某个 superpowers 技能时，在 `global/skills/` 提供同名的 harness 版并更新本表，插件其余技能不再被路由。

## 红线

- 档位未命中时不得调用被裁剪的技能，也不得以“先调用看看”绕过。
- 不得在澄清问题之前机械调用技能消耗澄清轮次。
- 技能流程与 harness 规则冲突时，按上述优先级处理，优先 harness 规则。
- 用户显式指定的 Skill 必须执行；超出配额的处理见规则优先级第 2 条。