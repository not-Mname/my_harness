---
name: using-superpowers
description: 在开始任何对话时使用——确立如何查找和使用技能，要求在任何响应（包括澄清性问题）之前调用 Skill 工具
version: "1.0.0"
license: MIT
metadata:
  hermes:
    tags: [meta, getting-started]
---

<SUBAGENT-STOP>
如果你是作为子智能体被分派来执行特定任务的，忽略此技能。
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
本技能已按 my_harness 工作流适配：技能调用由 Workflow 档位和档位裁剪表决定，不再采用“1% 可能适用就必须调用”的机制。

一个技能是否适用于你的任务，由当前档位和裁剪表决定；档位未命中时不调用该技能。
</EXTREMELY-IMPORTANT>

## 规则

**先确定档位，再按档位裁剪表调用技能。** 开始任务前先读取 `~/.codex/rules/workflow-policy.md` 确定当前档位（默认自动模式，输出 `[自动模式 -> 档位]`）；只有裁剪表命中的技能才必须调用。澄清性问题、探索代码库、查看文件不属于技能调用，不消耗技能流程。

**进入计划/设计阶段之前：** 按档位决定是否头脑风暴——轻量级压缩为必要澄清与简短方案确认，中量级做关键设计确认，重量级完整执行 brainstorming。

然后宣布"使用 [技能] 来 [目的]"，并严格遵循该技能。如果它有检查清单，为每个条目创建一个待办。

## 技能优先级

当多个技能都适用时，流程技能优先——它们决定处理方式，然后由实现技能（前端设计等）负责执行。头脑风暴和系统化调试是 Superpowers 中最常见的流程技能，但这条规则适用于任何流程技能。

- "让我们构建 X" → 先用 superpowers:brainstorming，再用实现技能。
- "修复这个 bug" → 先用 superpowers:systematic-debugging，再用领域技能。

## 档位裁剪表（my_harness）

本表定义各技能在不同 Workflow 档位下的调用深度；未命中的档位不调用对应技能。与 `~/.codex/rules/workflow-policy.md` 冲突时以策略为准。

| 技能 | 轻量级 | 中量级 | 重量级 |
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

对应技能的详细分级门禁见各技能文件；本表是路由入口。

## 红线

这些想法意味着停下——你在合理化：

| 想法 | 现实 |
|------|------|
| "这只是一个简单的问题" | 问题就是任务。检查技能。 |
| "我需要先了解更多上下文" | 技能检查在澄清性问题之前。 |
| "让我先探索一下代码库" | 技能告诉你如何探索。先检查。 |
| "我可以快速查一下 git/文件" | 文件缺少对话上下文。检查技能。 |
| "让我先收集信息" | 技能告诉你如何收集信息。 |
| "这不需要正式的技能" | 如果技能存在，就使用它。 |
| "我记得这个技能" | 技能会迭代更新。阅读当前版本。 |
| "这不算一个任务" | 行动 = 任务。检查技能。 |
| "技能太小题大做了" | 简单的事会变复杂。使用它。 |
| "让我先做这一件事" | 在做任何事之前先检查。 |
| "这样做感觉很高效" | 无纪律的行动浪费时间。技能防止这一点。 |
| "我知道那是什么意思" | 知道概念 ≠ 使用技能。调用它。 |

## 平台适配

如果你的运行环境在下面列出，请阅读对应的参考文件获取特殊说明：

- Codex：`references/codex-tools.md`
- Pi：`references/pi-tools.md`
- Copilot CLI：`references/copilot-tools.md`
- Hermes Agent：`references/hermes-tools.md`
- Qoder：`references/qoder-tools.md`

Gemini CLI 用户通过 GEMINI.md 自动获得 `references/gemini-tools.md` 的工具映射。

## 中国特色技能路由

当检测到以下场景时，**必须**优先调用对应的中国特色技能：

| 场景 | 调用技能 |
|------|---------|
| 代码审查且团队使用中文沟通 | **superpowers:chinese-code-review** |
| 使用 Gitee/Coding/极狐 GitLab | **superpowers:chinese-git-workflow** |
| 编写中文技术文档或 README | **superpowers:chinese-documentation** |
| 编写 git commit message（中文项目） | **superpowers:chinese-commit-conventions** |
| 构建 MCP 服务器/工具 | **superpowers:mcp-builder** |

**判断依据：**
- 项目中有中文注释、中文 README、或 .gitee 目录 → 启用中文系列技能
- commit 历史中有中文 → 使用中文提交规范
- 用户用中文交流 → 所有输出使用中文，优先考虑中国特色技能

中国特色技能与翻译技能**叠加使用**，不互斥。例如：做代码审查时，同时使用 requesting-code-review（流程）+ chinese-code-review（风格）。

## 用户指令

用户指令（CLAUDE.md、AGENTS.md、GEMINI.md 等、直接请求）优先于技能，技能又优先于默认行为。只有当你的人类伙伴明确告诉你跳过时，才能跳过技能工作流或指令。
