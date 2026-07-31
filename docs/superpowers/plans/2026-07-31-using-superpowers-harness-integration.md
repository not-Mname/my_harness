# using-superpowers harness 化与技能裁剪 实施计划

> **面向 AI 代理的工作者：** 由主 Agent 直接按步骤执行；步骤使用复选框（`- [ ]`）跟踪进度。本计划全部产物为 Markdown 技能/文档，无代码与测试步骤。

**目标：** 消除插件 `using-superpowers` 与本 harness 分档预算策略的冲突，为逐步与 superpowers 解耦铺路。

**架构：** 以“档位决定流程、规则优先于技能”为核心。新建 harness 版 `using-superpowers` 作为唯一技能路由入口（档位裁剪表 + 规则优先级 + 澄清协议），并把 5 个高冲突技能复制为 harness 裁剪版（前置档位门禁、改写冲突条款）。插件原版保持不动。

**技术栈：** Markdown skills（SKILL.md frontmatter），无代码。

---

## 文件结构

- 创建 `global/skills/using-superpowers/SKILL.md` — 技能路由唯一入口
- 创建 `global/skills/brainstorming/SKILL.md` — 头脑风暴裁剪版
- 创建 `global/skills/test-driven-development/SKILL.md` — TDD 裁剪版
- 创建 `global/skills/using-git-worktrees/SKILL.md` — worktree 裁剪版
- 创建 `global/skills/dispatching-parallel-agents/SKILL.md` — 并行派发裁剪版
- 创建 `global/skills/subagent-driven-development/SKILL.md` — SDD 裁剪版
- 创建 `docs/superpowers/plans/2026-07-31-using-superpowers-harness-integration.md` — 本计划

## 任务 1：保存实施计划

- [ ] 创建本计划文档
- [ ] Commit（`docs: 添加 using-superpowers harness 化实施计划`）

## 任务 2：harness 版 using-superpowers 路由技能

**文件：** 创建 `global/skills/using-superpowers/SKILL.md`

- [ ] 创建技能，包含：第一步定档位（读 `~/.codex/rules/workflow-policy.md`）、档位裁剪表、harness 规则优先级、澄清与假设协议、平台适配（Codex 工具映射）、解耦声明、红线
- [ ] 校验：全文无“1% 规则”式强制措辞；档位表与 workflow-policy 裁剪条款一致；声明取代插件同名技能
- [ ] Commit（`feat(skills): 添加 harness 版 using-superpowers 路由技能`）

## 任务 3：裁剪 brainstorming

**文件：** 创建 `global/skills/brainstorming/SKILL.md`

- [ ] 创建技能：档位门禁（轻量压缩为澄清+简短方案确认且不写 spec；中量合并澄清且 spec 可选；重量完整流程）、HARD-GATE 改为“确认后才实现”、核心流程（澄清→方案→展示设计→批准→spec→自检→用户审查→writing-plans）
- [ ] 校验：与 workflow-policy 轻/中/重裁剪条款一致
- [ ] Commit（`feat(skills): 添加 brainstorming 裁剪版`）

## 任务 4：裁剪 test-driven-development

**文件：** 创建 `global/skills/test-driven-development/SKILL.md`

- [ ] 创建技能：档位门禁（轻量不强制 TDD、测试轮给最高信息量验证；中量高风险优先；重量完整）、红-绿-重构核心流程、测试轮次配额口径、例外清单
- [ ] 校验：与 workflow-policy 测试配额（1/3 轮）一致
- [ ] Commit（`feat(skills): 添加 test-driven-development 裁剪版`）

## 任务 5：裁剪 using-git-worktrees

**文件：** 创建 `global/skills/using-git-worktrees/SKILL.md`

- [ ] 创建技能：默认禁止（对齐 `git-development.md` 的“未经允许不准创建 worktree/branch/tag”）、HITL 授权门禁、授权后的流程（检测隔离→原生工具→git 回退→忽略验证→设置→基线测试）
- [ ] 校验：未授权路径明确为“当前目录原地工作”
- [ ] Commit（`feat(skills): 添加 using-git-worktrees 裁剪版`）

## 任务 6：裁剪 dispatching-parallel-agents

**文件：** 创建 `global/skills/dispatching-parallel-agents/SKILL.md`

- [ ] 创建技能：派发前必读 workflow-policy + multi-agent-assistance、档位配额（轻量 1/3、中量 3/5）、并行独立条件、禁止并行清单、身份冻结、汇合核对
- [ ] 校验：与 workflow-policy 派发约束逐条一致
- [ ] Commit（`feat(skills): 添加 dispatching-parallel-agents 裁剪版`）

## 任务 7：裁剪 subagent-driven-development

**文件：** 创建 `global/skills/subagent-driven-development/SKILL.md`

- [ ] 创建技能：档位派发配额门禁、串行实施+评审约束（严禁同时派发多个实施 Agent）、并行写入改用 DPA、计划审查、四状态处理、收尾规则
- [ ] 校验：与 workflow-policy 的 SDD 条款一致（不并行实施、可创建范围明确的普通提交、不 push）
- [ ] Commit（`feat(skills): 添加 subagent-driven-development 裁剪版`）

## 任务 8：同步全局配置

- [ ] 复制 `global/skills/` 下 6 个目录到 `C:\Users\wjf\.codex\skills\`
- [ ] 校验：`~/.codex/skills/` 下 6 个技能就位；插件目录未改动
- [ ] Commit（`chore: 同步全局技能配置`，仅包含 global/ 变更）

## 任务 9：验证与收尾

- [ ] 校验 6 个新技能 frontmatter 合法、description 可检索、无残留占位符
- [ ] 冲突清单逐项回归：轻量级任务不触发 brainstorming/TDD；worktree 未授权不执行；派发遵守配额
- [ ] 更新 `docs/guides/workflow-modes.md` 补“技能裁剪”一节（可选）
- [ ] 最终 Commit（不 push）