---
name: workflow-heavy
description: Use when 用户显式调用 $workflow-heavy，要求将当前 Codex 对话持续锁定为完整执行适用 Superpowers 的重量级流程模式
---

# 持续重量级流程

开始任务前必须读取 `~/.codex/rules/workflow-policy.md`，并按重量级预算规划整个任务。

1. 显式调用后持续锁定重量级，不再自动判断任务档位。
2. 调用消息包含任务正文时，切换后立即按重量级执行；没有任务正文时只确认切换结果。
3. 最新显式模式切换优先，任务中途切换只影响未完成步骤。
4. 不自动升级或降级；只有不可绕过底线冲突时才进入 HITL。
