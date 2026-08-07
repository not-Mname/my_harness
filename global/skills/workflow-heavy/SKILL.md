---
name: workflow-heavy
description: Use when 用户显式调用 $workflow-heavy，要求将当前 Codex 对话持续锁定为完整执行适用 Superpowers 的重量级流程模式
---

# 持续重量级流程

开始任务前必须读取 `~/.codex/rules/workflow-policy.md`，并按重量级预算规划整个任务。

重量级任务必须先进入 `superpowers:using-superpowers` 总入口，再按其裁剪表执行所有适用的 Superpowers 流程。不得以“已选择重量级”代替实际技能调用；在完成 brainstorming、writing-plans、test-driven-development、verification-before-completion、requesting-code-review 和 finishing-a-development-branch 等适用阶段前，不得宣布任务完成。

执行顺序：确定适用技能 -> 宣布并调用流程技能 -> 完成每个技能的门禁和验证 -> 汇总评审与收尾证据。若某阶段不适用，必须记录不适用的理由。

1. 显式调用后持续锁定重量级，不再自动判断任务档位。
2. 调用消息包含任务正文时，切换后立即按重量级执行；没有任务正文时只确认切换结果。
3. 最新显式模式切换优先，任务中途切换只影响未完成步骤。
4. 不自动升级或降级；只有不可绕过底线冲突时才进入 HITL。
